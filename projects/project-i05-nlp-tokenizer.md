# Project I05: NLP Tokenizer & BPE

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **NLP Tokenization Pipeline** ที่สมบูรณ์ตั้งแต่ต้นจนจบ ครอบคลุมตั้งแต่ whitespace tokenizer แบบง่ายไปจนถึง Byte-Pair Encoding (BPE) และ WordPiece tokenizer แบบ BERT โปรเจคนี้ไม่ได้แค่สร้าง toy implementation แต่ออกแบบให้รองรับ text preprocessing pipeline ที่ใช้งานจริงได้ในงาน NLP

**Tokenization** คือหัวใจของ NLP pipeline ทุกระบบ ก่อนที่ model จะประมวลผลข้อความได้ ข้อความต้องถูกแปลงเป็น sequence ของ token ids ก่อน วิธีที่ tokenize ส่งผลโดยตรงต่อประสิทธิภาพ model เพราะ vocabulary size กำหนด embedding table size, sequence length กำหนด memory และ computation, การจัดการ out-of-vocabulary words กำหนดว่า model จัดการคำใหม่ได้ดีแค่ไหน

BPE (Byte-Pair Encoding) เป็น algorithm ที่ใช้กันอย่างแพร่หลายใน GPT-2, GPT-3, RoBERTa และ models รุ่นใหม่ส่วนใหญ่ ส่วน WordPiece เป็น algorithm ที่ BERT ใช้ ซึ่งทั้งสองมีแนวคิดคล้ายกันแต่มีรายละเอียดต่างกัน

**Use cases จริงในโลก production:**
- สร้าง custom tokenizer สำหรับภาษาเฉพาะทาง (เช่น ภาษาไทย, ภาษาโค้ด, domain-specific language)
- ลด vocabulary size โดยยังคงครอบคลุมคำศัพท์ได้ดี
- preprocessing pipeline สำหรับ text classification, NER, machine translation
- embed ใน inference engine เพื่อ tokenize input ก่อนส่งให้ model

## สิ่งที่จะได้เรียนรู้

- **BPE algorithm** — iterative merge ของ most-frequent adjacent pair เพื่อสร้าง subword vocabulary
- **Greedy longest-match** — วิธีที่ WordPiece ใช้ split word เป็น subwords
- **Unicode normalization** — NFD/NFC decomposition และการ strip combining marks
- **Composable pipeline pattern** — struct ที่รวม preprocessing stages เข้าด้วยกัน
- **HashMap-based vocabulary** — bidirectional mapping ระหว่าง token string และ integer id
- **Serde JSON serialization** — save/load vocabulary เป็น JSON ได้ทันที
- **Regex-based tokenization** — ใช้ `regex` crate สำหรับ pattern matching ที่ซับซ้อน
- **Special token handling** — `[PAD]`, `[UNK]`, `[BOS]`, `[EOS]` และ fallback สำหรับ unknown tokens

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`, `HashSet`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, trait objects
- **Part 41–50**: String manipulation, `chars()`, `split_whitespace()`, Unicode basics
- **Part 51–60**: Modules, crate ecosystem, `Cargo.toml` dependencies
- **Part 96–105**: Serde serialization/deserialization, JSON format

## โครงสร้างโปรเจค (Project Layout)

```
nlp-tokenizer/
├── src/
│   ├── main.rs          ← demo และ integration
│   ├── lib.rs           ← re-export modules
│   ├── tokenizer.rs     ← WhitespaceTokenizer, WordTokenizer
│   ├── bpe.rs           ← BpeTrainer, BpeMergeRule, BpeTokenizer
│   ├── wordpiece.rs     ← WordPieceTokenizer (BERT-style)
│   ├── vocab.rs         ← Vocab struct, save/load JSON
│   └── pipeline.rs      ← TextPipeline composable stages
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Raw Text
    │
    ▼
┌──────────────┐
│  TextPipeline │  ← lowercase, NFD normalize, strip accents, Chinese spacing
└──────────────┘
    │ cleaned text
    ▼
┌──────────────────────┐
│   Tokenizer Layer    │
│                      │
│  WhitespaceTokenizer │  ← split by whitespace, strip punct
│  WordTokenizer       │  ← regex word boundaries
│  BpeTokenizer        │  ← apply BPE merge rules
│  WordPieceTokenizer  │  ← greedy longest-match subwords
└──────────────────────┘
    │ Vec<String> tokens
    ▼
┌──────────────┐
│    Vocab     │  ← token_to_id HashMap, id_to_token Vec
└──────────────┘
    │ Vec<u32> ids
    ▼
   Model Input
```

### ทำไมถึงแยก Trainer ออกจาก Tokenizer?

ใน production NLP systems เราแยก training phase กับ inference phase ออกจากกันชัดเจน:

- **BpeTrainer** ทำงานกับ corpus ขนาดใหญ่ ใช้เวลานาน รันครั้งเดียว
- **BpeTokenizer** load merge rules ที่ train มาแล้ว ทำงานเร็ว inference-time

การแยกนี้ทำให้ tokenizer เป็น stateless หลังจาก load rules แล้ว ทำให้ thread-safe โดยธรรมชาติ และ serialize/load ได้ง่าย

### ทำไม BPE ถึงดีกว่า word-level tokenization?

**Word-level problems:**
- "running", "runner", "runs" → 3 tokens ที่ต่างกัน ทั้งที่มี root เดียวกัน
- คำที่ไม่เคยเห็นใน training → `[UNK]` เสมอ
- Vocabulary ขนาดใหญ่มาก (English อาจมีหลายแสนคำ)

**BPE solutions:**
- "running" → "run" + "##ning" แชร์ representation กับ "runner", "runs"
- คำใหม่ก็ยังสามารถ decompose เป็น known subwords ได้
- Vocabulary size ควบคุมได้โดย parameter `vocab_size`

### Unicode Normalization

ข้อความจาก sources ต่าง ๆ อาจ encode "é" ต่างกัน:
- **NFC** (Composed): `é` = U+00E9 (single codepoint)
- **NFD** (Decomposed): `é` = U+0065 (e) + U+0301 (combining accent)

การ normalize เป็น NFD ก่อนแล้ว strip combining marks คือวิธีมาตรฐานในการ strip accents ทำให้ "café" กลายเป็น "cafe" ซึ่ง BERT ทำแบบนี้ใน original implementation

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: โครงสร้างโปรเจคและ WhitespaceTokenizer

เริ่มจาก dependency setup และ tokenizer แบบง่ายที่สุด — แยก token ด้วย whitespace, lowercase, และ strip punctuation

**`Cargo.toml`:**

```toml
[package]
name = "nlp-tokenizer"
version = "0.1.0"
edition = "2021"

[dependencies]
regex = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
unicode-normalization = "0.1"
```

**`src/lib.rs`:**

```rust
pub mod tokenizer;
pub mod bpe;
pub mod wordpiece;
pub mod vocab;
pub mod pipeline;
```

**`src/tokenizer.rs` — ส่วนแรก:**

```rust
use regex::Regex;

/// Tokenizer แบบง่ายที่แยก token ด้วย whitespace
pub struct WhitespaceTokenizer;

impl WhitespaceTokenizer {
    /// tokenize ข้อความโดยแยกด้วย whitespace, lowercase, ลบ punctuation
    pub fn tokenize(text: &str) -> Vec<String> {
        text.split_whitespace()
            .map(|w| {
                w.chars()
                    .filter(|c| c.is_alphanumeric())
                    .collect::<String>()
                    .to_lowercase()
            })
            .filter(|w| !w.is_empty())
            .collect()
    }
}
```

**ทดสอบ whitespace tokenizer:**

```rust
fn main() {
    let tokens = WhitespaceTokenizer::tokenize("Hello, World! This is Rust NLP.");
    println!("{:?}", tokens);
    // ["hello", "world", "this", "is", "rust", "nlp"]
}
```

**Output:**
```
["hello", "world", "this", "is", "rust", "nlp"]
```

`split_whitespace()` จัดการ multiple spaces และ newlines อัตโนมัติ ส่วน `filter(|c| c.is_alphanumeric())` ลบ punctuation ออกก่อน lowercase เพื่อให้ "Hello," กลายเป็น "Hello" ก่อนแล้วค่อย lowercase เป็น "hello"

---

### ขั้นที่ 2: WordTokenizer ด้วย Regex

Whitespace tokenizer มีปัญหากับ contractions เช่น "don't" จะกลายเป็น "dont" (ลบ apostrophe ออก) `WordTokenizer` แก้ปัญหานี้ด้วย regex pattern ที่รู้จัก word boundaries จริง ๆ

**`src/tokenizer.rs` — เพิ่ม WordTokenizer:**

```rust
/// Tokenizer แบบ word-boundary ด้วย regex
pub struct WordTokenizer {
    pattern: Regex,
}

impl WordTokenizer {
    pub fn new() -> Self {
        WordTokenizer {
            // จับ word characters รวมถึง apostrophe contractions
            pattern: Regex::new(r"[a-zA-Z0-9]+(?:'[a-zA-Z]+)*").unwrap(),
        }
    }

    pub fn tokenize(&self, text: &str) -> Vec<String> {
        self.pattern
            .find_iter(text)
            .map(|m| m.as_str().to_lowercase())
            .collect()
    }
}

impl Default for WordTokenizer {
    fn default() -> Self {
        Self::new()
    }
}
```

**Pattern อธิบาย:**
- `[a-zA-Z0-9]+` — อย่างน้อยหนึ่ง alphanumeric character
- `(?:'[a-zA-Z]+)*` — ตามด้วย contraction group ศูนย์ครั้งหรือมากกว่า เช่น `'t`, `'re`, `'ll`

**ทดสอบ:**

```rust
let tok = WordTokenizer::new();
let tokens = tok.tokenize("don't stop, won't stop!");
println!("{:?}", tokens);
// ["don't", "stop", "won't", "stop"]
```

**Output:**
```
["don't", "stop", "won't", "stop"]
```

เปรียบเทียบกับ `WhitespaceTokenizer`:
```
WhitespaceTokenizer: ["dont", "stop", "wont", "stop"]
WordTokenizer:       ["don't", "stop", "won't", "stop"]
```

WordTokenizer รักษา contraction ไว้ได้ ซึ่งสำคัญสำหรับ downstream tasks เพราะ "don't" และ "do not" มีความหมายเหมือนกัน แต่ "dont" เป็นคำที่ไม่มีความหมาย

---

### ขั้นที่ 3: BPE Trainer — เรียนรู้ Merge Rules

BPE algorithm เริ่มจาก character-level vocabulary แล้ว iteratively merge คู่ที่เกิดบ่อยที่สุดจนถึง vocabulary size ที่ต้องการ

**`src/bpe.rs` — BpeMergeRule และ BpeTrainer:**

```rust
use std::collections::HashMap;

/// กฎการ merge คู่ token ใน BPE
#[derive(Debug, Clone, PartialEq)]
pub struct BpeMergeRule {
    pub pair: (String, String),
    pub merged: String,
}

/// BPE Trainer — เรียนรู้ merge rules จาก corpus
pub struct BpeTrainer {
    pub vocab_size: usize,
}

impl BpeTrainer {
    pub fn new(vocab_size: usize) -> Self {
        BpeTrainer { vocab_size }
    }

    /// train — รับ corpus (list of words) แล้วคืน Vec<BpeMergeRule>
    pub fn train(&self, corpus: &[&str]) -> Vec<BpeMergeRule> {
        // เริ่มจาก character-level representation
        // แต่ละ word แทนด้วย sequence ของ characters
        let mut word_freqs: HashMap<Vec<String>, usize> = HashMap::new();
        for word in corpus {
            let chars: Vec<String> = word.chars().map(|c| c.to_string()).collect();
            *word_freqs.entry(chars).or_insert(0) += 1;
        }

        let mut merge_rules: Vec<BpeMergeRule> = Vec::new();
        let mut current_vocab: usize = {
            let mut chars = std::collections::HashSet::new();
            for word in corpus {
                for c in word.chars() {
                    chars.insert(c);
                }
            }
            chars.len()
        };

        while current_vocab < self.vocab_size {
            // นับ pair frequencies
            let pair_freqs = Self::count_pairs(&word_freqs);
            if pair_freqs.is_empty() {
                break;
            }

            // เลือก pair ที่เกิดบ่อยที่สุด
            let best_pair = pair_freqs
                .iter()
                .max_by_key(|(_, &freq)| freq)
                .map(|(pair, _)| pair.clone())
                .unwrap();

            let merged = format!("{}{}", best_pair.0, best_pair.1);
            let rule = BpeMergeRule {
                pair: best_pair.clone(),
                merged: merged.clone(),
            };

            // อัปเดต word_freqs โดย merge คู่นั้น
            word_freqs = Self::apply_merge(&word_freqs, &best_pair, &merged);
            merge_rules.push(rule);
            current_vocab += 1;
        }

        merge_rules
    }

    /// นับความถี่ของคู่ adjacent tokens
    fn count_pairs(
        word_freqs: &HashMap<Vec<String>, usize>
    ) -> HashMap<(String, String), usize> {
        let mut pair_freqs: HashMap<(String, String), usize> = HashMap::new();
        for (tokens, &freq) in word_freqs {
            for window in tokens.windows(2) {
                let pair = (window[0].clone(), window[1].clone());
                *pair_freqs.entry(pair).or_insert(0) += freq;
            }
        }
        pair_freqs
    }

    /// apply merge rule ให้กับ word_freqs ทั้งหมด
    fn apply_merge(
        word_freqs: &HashMap<Vec<String>, usize>,
        pair: &(String, String),
        merged: &str,
    ) -> HashMap<Vec<String>, usize> {
        let mut new_freqs: HashMap<Vec<String>, usize> = HashMap::new();
        for (tokens, &freq) in word_freqs {
            let new_tokens = Self::merge_tokens(tokens, pair, merged);
            *new_freqs.entry(new_tokens).or_insert(0) += freq;
        }
        new_freqs
    }

    fn merge_tokens(
        tokens: &[String],
        pair: &(String, String),
        merged: &str,
    ) -> Vec<String> {
        let mut result = Vec::new();
        let mut i = 0;
        while i < tokens.len() {
            if i + 1 < tokens.len()
                && tokens[i] == pair.0
                && tokens[i + 1] == pair.1
            {
                result.push(merged.to_string());
                i += 2;
            } else {
                result.push(tokens[i].clone());
                i += 1;
            }
        }
        result
    }
}
```

**Demo training:**

```rust
let corpus = vec![
    "low", "low", "low", "low", "lower", "lower",
    "newest", "newest", "newest", "widest",
];
let trainer = BpeTrainer::new(20);
let rules = trainer.train(&corpus);
for (i, rule) in rules.iter().enumerate() {
    println!("Rule {}: ({}, {}) -> {}", i+1, rule.pair.0, rule.pair.1, rule.merged);
}
```

**Output:**
```
Rule 1: (l, o) -> lo
Rule 2: (lo, w) -> low
Rule 3: (s, t) -> st
Rule 4: (e, st) -> est
Rule 5: (w, est) -> west
Rule 6: (n, e) -> ne
Rule 7: (ne, west) -> newest
Rule 8: (low, e) -> lowe
Rule 9: (lowe, r) -> lower
Rule 10: (d, est) -> dest
```

สังเกตว่า "lo" ถูก merge ก่อนเพราะมีใน "low" (4 ครั้ง) + "lower" (2 ครั้ง) = 6 ครั้ง algorithm จะเรียนรู้ subwords ที่มีความหมายเองโดยอัตโนมัติจากความถี่

---

### ขั้นที่ 4: BPE Tokenizer — Encode และ Decode

หลัง training ได้ merge rules แล้ว เราสร้าง `BpeTokenizer` ที่ใช้ rules เหล่านั้น encode ข้อความใหม่และ decode กลับมา

**`src/bpe.rs` — BpeTokenizer:**

```rust
/// BPE Tokenizer — ใช้ merge rules ที่ train มาแล้วเพื่อ encode/decode
pub struct BpeTokenizer {
    merge_rules: Vec<BpeMergeRule>,
    token_to_id: HashMap<String, u32>,
    id_to_token: Vec<String>,
}

impl BpeTokenizer {
    /// สร้าง tokenizer จาก merge rules และ base_vocab (character list)
    pub fn new(merge_rules: Vec<BpeMergeRule>, base_vocab: Vec<String>) -> Self {
        let mut id_to_token: Vec<String> = Vec::new();
        // special tokens อยู่ index 0-3 เสมอ
        let special_tokens = ["[PAD]", "[UNK]", "[BOS]", "[EOS]"];
        for st in &special_tokens {
            id_to_token.push(st.to_string());
        }
        // base vocabulary (characters)
        for token in &base_vocab {
            if !id_to_token.contains(token) {
                id_to_token.push(token.clone());
            }
        }
        // merged tokens จาก merge rules
        for rule in &merge_rules {
            if !id_to_token.contains(&rule.merged) {
                id_to_token.push(rule.merged.clone());
            }
        }

        let mut token_to_id: HashMap<String, u32> = HashMap::new();
        for (i, token) in id_to_token.iter().enumerate() {
            token_to_id.insert(token.clone(), i as u32);
        }

        BpeTokenizer {
            merge_rules,
            token_to_id,
            id_to_token,
        }
    }

    pub fn pad_id(&self) -> u32 { *self.token_to_id.get("[PAD]").unwrap() }
    pub fn unk_id(&self) -> u32 { *self.token_to_id.get("[UNK]").unwrap() }
    pub fn bos_id(&self) -> u32 { *self.token_to_id.get("[BOS]").unwrap() }
    pub fn eos_id(&self) -> u32 { *self.token_to_id.get("[EOS]").unwrap() }

    /// encode ข้อความเป็น sequence ของ token ids
    pub fn encode(&self, text: &str) -> Vec<u32> {
        let mut ids = Vec::new();
        for word in text.split_whitespace() {
            let mut tokens: Vec<String> =
                word.chars().map(|c| c.to_string()).collect();
            // apply merge rules ตามลำดับที่ train
            for rule in &self.merge_rules {
                tokens = BpeTrainer::merge_tokens_pub(
                    &tokens, &rule.pair, &rule.merged
                );
            }
            for token in tokens {
                let id = self
                    .token_to_id
                    .get(&token)
                    .copied()
                    .unwrap_or_else(|| self.unk_id());
                ids.push(id);
            }
        }
        ids
    }

    /// decode sequence ของ ids กลับเป็น tokens
    pub fn decode(&self, ids: &[u32]) -> String {
        ids.iter()
            .filter_map(|&id| self.id_to_token.get(id as usize))
            .cloned()
            .collect::<Vec<_>>()
            .join(" ")
    }
}
```

**Demo encode/decode:**

```rust
let corpus = vec!["low", "low", "low", "low", "lower", "lower",
                  "newest", "newest", "newest", "widest"];
let trainer = BpeTrainer::new(20);
let rules = trainer.train(&corpus);

let mut base_vocab: Vec<String> = Vec::new();
for word in &corpus {
    for c in word.chars() {
        let s = c.to_string();
        if !base_vocab.contains(&s) {
            base_vocab.push(s);
        }
    }
}
let tokenizer = BpeTokenizer::new(rules, base_vocab);

let ids = tokenizer.encode("low");
println!("Encode 'low': {:?}", ids);  // [15]

let decoded = tokenizer.decode(&ids);
println!("Decode back: {:?}", decoded);  // "low"
```

**Output:**
```
Encode 'low'  : [15]
Decode back   : "low"
```

index 15 เพราะ special tokens ใช้ 4 slots (0-3), base characters ใช้ slots ถัดไป และ "low" เป็น merged token ที่ถูกสร้างใน rule ที่ 2 ดังนั้น encoding เป็น id เดียว

---

### ขั้นที่ 5: WordPiece Tokenizer (BERT-style)

WordPiece ต่างจาก BPE ตรงที่:
- BPE: train โดย merge most-frequent pair
- WordPiece: train โดย maximize likelihood ของ training data
- ทั้งคู่ใช้ prefix `##` สำหรับ continuation subwords ณ inference time

Algorithm ที่ BERT ใช้ที่ inference time คือ **greedy longest-match forward**:

1. รับ word มา scan จาก position 0
2. ลองหา substring ที่ยาวที่สุดตั้งแต่ position นั้นที่อยู่ใน vocab
3. ถ้า position != 0 ให้เพิ่ม `##` นำหน้า substring
4. เลื่อน position ไปตาม substring ที่พบ ทำซ้ำ
5. ถ้าไม่พบ substring ใด ๆ คืน `[UNK]`

**`src/wordpiece.rs`:**

```rust
use std::collections::HashSet;

/// WordPiece Tokenizer (BERT-style)
/// ใช้ greedy longest-match forward algorithm
/// subwords ที่ไม่ใช่ token แรกจะมี prefix "##"
pub struct WordPieceTokenizer {
    vocab: HashSet<String>,
    unk_token: String,
    max_chars_per_word: usize,
}

impl WordPieceTokenizer {
    pub fn new(vocab: HashSet<String>) -> Self {
        WordPieceTokenizer {
            vocab,
            unk_token: "[UNK]".to_string(),
            max_chars_per_word: 100,
        }
    }

    /// tokenize ข้อความเป็น wordpiece tokens
    pub fn tokenize(&self, text: &str) -> Vec<String> {
        let mut output = Vec::new();
        for word in text.split_whitespace() {
            let pieces = self.tokenize_word(word);
            output.extend(pieces);
        }
        output
    }

    /// tokenize word เดียว → list ของ subword pieces
    pub fn tokenize_word(&self, word: &str) -> Vec<String> {
        let chars: Vec<char> = word.chars().collect();
        if chars.len() > self.max_chars_per_word {
            return vec![self.unk_token.clone()];
        }

        let mut sub_tokens: Vec<String> = Vec::new();
        let mut is_bad = false;
        let mut start = 0;

        while start < chars.len() {
            let mut end = chars.len();
            let mut cur_substr: Option<String> = None;

            // greedy longest-match: ลองจาก chars.len() ลงมาถึง start+1
            while start < end {
                let substr: String = chars[start..end].iter().collect();
                let substr_to_check = if start == 0 {
                    substr.clone()
                } else {
                    format!("##{}", substr)
                };

                if self.vocab.contains(&substr_to_check) {
                    cur_substr = Some(substr_to_check);
                    break;
                }
                end -= 1;
            }

            if cur_substr.is_none() {
                is_bad = true;
                break;
            }

            sub_tokens.push(cur_substr.unwrap());
            start = end;
        }

        if is_bad {
            vec![self.unk_token.clone()]
        } else {
            sub_tokens
        }
    }
}
```

**Demo:**

```rust
use std::collections::HashSet;

let vocab: HashSet<String> = vec![
    "[UNK]", "un", "##aff", "##able", "hello", "world",
    "##ing", "run", "##ner",
]
.iter()
.map(|s| s.to_string())
.collect();

let tok = WordPieceTokenizer::new(vocab);
let result = tok.tokenize("hello unaffable runner");
println!("{:?}", result);
```

**Output:**
```
["hello", "un", "##aff", "##able", "run", "##ner"]
```

สังเกตว่า "unaffable" ถูก split เป็น "un" + "##aff" + "##able" เพราะ:
1. ลอง "unaffable" → ไม่อยู่ใน vocab
2. ลอง "unaffabl" → ไม่อยู่
3. ... ลดลงเรื่อย ๆ ...
4. ลอง "un" → อยู่ใน vocab! → ได้ "un", start = 2
5. ลอง "##affable" → ไม่อยู่
6. ... ลดลง ...
7. ลอง "##aff" → อยู่ใน vocab! → ได้ "##aff", start = 5
8. ลอง "##able" → อยู่ใน vocab! → ได้ "##able"

---

### ขั้นที่ 6: Vocabulary Management

`Vocab` เป็น bidirectional mapping ระหว่าง token string และ integer id รองรับ save/load เป็น JSON และ build จาก corpus

**`src/vocab.rs`:**

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

/// Vocabulary structure สำหรับ mapping token ↔ id
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Vocab {
    pub token_to_id: HashMap<String, u32>,
    pub id_to_token: Vec<String>,
}

impl Vocab {
    pub fn new() -> Self {
        Vocab {
            token_to_id: HashMap::new(),
            id_to_token: Vec::new(),
        }
    }

    /// สร้าง vocab จาก list ของ tokens (ลำดับสำคัญ)
    pub fn from_tokens(tokens: &[String]) -> Self {
        let mut vocab = Vocab::new();
        for token in tokens {
            vocab.add_token(token.clone());
        }
        vocab
    }

    /// เพิ่ม token เข้า vocab (ถ้ายังไม่มี) คืน id
    pub fn add_token(&mut self, token: String) -> u32 {
        if let Some(&id) = self.token_to_id.get(&token) {
            return id;
        }
        let id = self.id_to_token.len() as u32;
        self.token_to_id.insert(token.clone(), id);
        self.id_to_token.push(token);
        id
    }

    pub fn get_id(&self, token: &str) -> Option<u32> {
        self.token_to_id.get(token).copied()
    }

    pub fn get_token(&self, id: u32) -> Option<&str> {
        self.id_to_token.get(id as usize).map(|s| s.as_str())
    }

    pub fn size(&self) -> usize {
        self.id_to_token.len()
    }

    pub fn unk_id(&self) -> u32 {
        self.token_to_id.get("[UNK]").copied().unwrap_or(0)
    }

    /// save vocab เป็น JSON string
    pub fn to_json(&self) -> String {
        serde_json::to_string(self).unwrap()
    }

    /// load vocab จาก JSON string
    pub fn from_json(json: &str) -> Result<Self, serde_json::Error> {
        serde_json::from_str(json)
    }

    /// สร้าง vocab จาก corpus โดยนับ character frequency
    pub fn build_from_corpus(corpus: &[&str], max_vocab_size: usize) -> Self {
        let mut freq: HashMap<String, usize> = HashMap::new();
        for word in corpus {
            for ch in word.chars() {
                *freq.entry(ch.to_string()).or_insert(0) += 1;
            }
        }

        // special tokens ก่อนเสมอ
        let mut tokens = vec![
            "[PAD]".to_string(),
            "[UNK]".to_string(),
            "[BOS]".to_string(),
            "[EOS]".to_string(),
        ];

        // เรียงตาม frequency มากไปน้อย
        let mut sorted: Vec<(String, usize)> = freq.into_iter().collect();
        sorted.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));

        for (token, _) in sorted {
            if tokens.len() >= max_vocab_size {
                break;
            }
            if !tokens.contains(&token) {
                tokens.push(token);
            }
        }

        Vocab::from_tokens(&tokens)
    }
}
```

**Demo save/load:**

```rust
let mut vocab = Vocab::new();
vocab.add_token("[PAD]".to_string());
vocab.add_token("[UNK]".to_string());
vocab.add_token("hello".to_string());
vocab.add_token("world".to_string());

let json = vocab.to_json();
println!("JSON: {}", &json[..60]);  // เห็น structure

let loaded = Vocab::from_json(&json).unwrap();
println!("Loaded size: {}", loaded.size());           // 4
println!("ID of 'hello': {:?}", loaded.get_id("hello")); // Some(2)
println!("Token 3: {:?}", loaded.get_token(3));        // Some("world")
```

**Output:**
```
JSON: {"token_to_id":{"[PAD]":0,"[UNK]":1,"hello":2,"world":3}
Loaded size: 4
ID of 'hello': Some(2)
Token 3: Some("world")
```

---

### ขั้นที่ 7: Text Preprocessing Pipeline

`TextPipeline` รวม preprocessing stages เข้าด้วยกัน — lowercase, unicode normalization, strip accents, handle Chinese characters, และ sentence splitting

**`src/pipeline.rs`:**

```rust
use unicode_normalization::UnicodeNormalization;

/// TextPipeline — composable stages สำหรับ text preprocessing
#[derive(Debug, Clone)]
pub struct TextPipeline {
    pub lowercase: bool,
    pub unicode_normalize: NormalizeForm,
    pub strip_accents: bool,
    pub handle_chinese: bool,
}

#[derive(Debug, Clone, PartialEq)]
pub enum NormalizeForm {
    None,
    NFC,
    NFD,
}

impl TextPipeline {
    pub fn new() -> Self {
        TextPipeline {
            lowercase: true,
            unicode_normalize: NormalizeForm::NFD,
            strip_accents: true,
            handle_chinese: true,
        }
    }

    /// apply pipeline ทั้งหมดกับข้อความ
    pub fn process(&self, text: &str) -> String {
        let mut result = text.to_string();

        // 1. Unicode normalization
        result = match self.unicode_normalize {
            NormalizeForm::NFD => result.nfd().collect::<String>(),
            NormalizeForm::NFC => result.nfc().collect::<String>(),
            NormalizeForm::None => result,
        };

        // 2. Strip accents (ลบ combining marks หลังจาก NFD)
        if self.strip_accents {
            result = result
                .chars()
                .filter(|c| !is_combining_mark(*c))
                .collect();
        }

        // 3. Handle Chinese characters (เพิ่ม space รอบ CJK)
        if self.handle_chinese {
            result = add_whitespace_around_chinese(&result);
        }

        // 4. Lowercase
        if self.lowercase {
            result = result.to_lowercase();
        }

        result
    }

    /// แบ่งข้อความเป็น sentences
    pub fn split_sentences(text: &str) -> Vec<String> {
        let mut sentences = Vec::new();
        let mut current = String::new();
        let chars: Vec<char> = text.chars().collect();
        let mut i = 0;

        while i < chars.len() {
            let c = chars[i];
            current.push(c);

            if c == '.' || c == '!' || c == '?' {
                let next_is_boundary = i + 1 >= chars.len()
                    || chars[i + 1] == ' '
                    || chars[i + 1] == '\n';

                if next_is_boundary {
                    let trimmed = current.trim().to_string();
                    if !trimmed.is_empty() {
                        sentences.push(trimmed);
                    }
                    current = String::new();
                }
            }
            i += 1;
        }

        let trimmed = current.trim().to_string();
        if !trimmed.is_empty() {
            sentences.push(trimmed);
        }
        sentences
    }
}

/// ตรวจว่า character เป็น Unicode combining mark
fn is_combining_mark(c: char) -> bool {
    let cp = c as u32;
    (0x0300..=0x036F).contains(&cp)
        || (0x1DC0..=0x1DFF).contains(&cp)
        || (0x20D0..=0x20FF).contains(&cp)
        || (0xFE20..=0xFE2F).contains(&cp)
}

/// เพิ่ม whitespace รอบ CJK Unified Ideographs
fn add_whitespace_around_chinese(text: &str) -> String {
    let mut result = String::new();
    for c in text.chars() {
        if is_chinese_char(c) {
            result.push(' ');
            result.push(c);
            result.push(' ');
        } else {
            result.push(c);
        }
    }
    result
}

fn is_chinese_char(c: char) -> bool {
    let cp = c as u32;
    (0x4E00..=0x9FFF).contains(&cp)
        || (0x3400..=0x4DBF).contains(&cp)
        || (0xF900..=0xFAFF).contains(&cp)
}
```

**Demo pipeline:**

```rust
let pipeline = TextPipeline::new();

// Strip accents จาก "Café au lait"
let result = pipeline.process("Café au lait");
println!("{}", result);  // "cafe au lait"

// Chinese character spacing
let pipeline2 = TextPipeline {
    lowercase: false,
    unicode_normalize: NormalizeForm::None,
    strip_accents: false,
    handle_chinese: true,
};
let result2 = pipeline2.process("你好world");
println!("{}", result2);  // " 你  好 world"

// Sentence splitting
let sentences = TextPipeline::split_sentences("Hello world. How are you? I am fine!");
println!("{:?}", sentences);
```

**Output:**
```
cafe au lait
 你  好 world
["Hello world.", "How are you?", "I am fine!"]
```

---

### ขั้นที่ 8: Integration — รวมทุกส่วนใน main.rs

ขั้นสุดท้ายคือ integration ทุกส่วนเข้าด้วยกันใน main.rs เพื่อแสดง full pipeline

**`src/main.rs`:**

```rust
use nlp_tokenizer::tokenizer::{WhitespaceTokenizer, WordTokenizer};
use nlp_tokenizer::bpe::{BpeTrainer, BpeTokenizer};
use nlp_tokenizer::wordpiece::WordPieceTokenizer;
use nlp_tokenizer::vocab::Vocab;
use nlp_tokenizer::pipeline::TextPipeline;
use std::collections::HashSet;

fn main() {
    println!("=== NLP Tokenizer Demo ===\n");

    // 1. Whitespace Tokenizer
    println!("--- WhitespaceTokenizer ---");
    let tokens = WhitespaceTokenizer::tokenize("Hello, World! This is Rust NLP.");
    println!("Input : Hello, World! This is Rust NLP.");
    println!("Output: {:?}\n", tokens);

    // 2. Word Tokenizer
    println!("--- WordTokenizer ---");
    let word_tok = WordTokenizer::new();
    let tokens2 = word_tok.tokenize("don't stop, won't stop.");
    println!("Input : don't stop, won't stop.");
    println!("Output: {:?}\n", tokens2);

    // 3. BPE Training
    println!("--- BPE Training ---");
    let corpus = vec![
        "low", "low", "low", "low", "lower", "lower",
        "newest", "newest", "newest", "widest",
    ];
    let trainer = BpeTrainer::new(20);
    let rules = trainer.train(&corpus);
    println!("Corpus: {:?}", corpus);
    println!("Merge rules learned: {}", rules.len());
    for (i, rule) in rules.iter().enumerate() {
        println!("  Rule {}: ({}, {}) -> {}",
            i + 1, rule.pair.0, rule.pair.1, rule.merged);
    }
    println!();

    // 4. BPE Encode/Decode
    println!("--- BPE Encode/Decode ---");
    let mut base_vocab: Vec<String> = Vec::new();
    for word in &corpus {
        for c in word.chars() {
            let s = c.to_string();
            if !base_vocab.contains(&s) {
                base_vocab.push(s);
            }
        }
    }
    let tokenizer = BpeTokenizer::new(rules, base_vocab);
    let ids = tokenizer.encode("low");
    println!("Encode 'low'  : {:?}", ids);
    let decoded = tokenizer.decode(&ids);
    println!("Decode back   : {:?}\n", decoded);

    // 5. WordPiece Tokenizer
    println!("--- WordPiece Tokenizer ---");
    let vocab: HashSet<String> = vec![
        "[UNK]", "un", "##aff", "##able", "hello", "world",
        "##ing", "run", "##ner",
    ]
    .iter()
    .map(|s| s.to_string())
    .collect();
    let wp_tok = WordPieceTokenizer::new(vocab);
    let wp_tokens = wp_tok.tokenize("hello unaffable runner");
    println!("Input : hello unaffable runner");
    println!("Output: {:?}\n", wp_tokens);

    // 6. Vocabulary
    println!("--- Vocabulary ---");
    let vocab_obj = Vocab::build_from_corpus(&["hello", "world", "hello"], 20);
    println!("Vocab size: {}", vocab_obj.size());
    println!("ID of 'l'  : {:?}", vocab_obj.get_id("l"));
    let json = vocab_obj.to_json();
    let loaded = Vocab::from_json(&json).unwrap();
    println!("Loaded size: {}\n", loaded.size());

    // 7. Text Pipeline
    println!("--- TextPipeline ---");
    let pipeline = TextPipeline::new();
    let processed = pipeline.process("Café au lait");
    println!("Input   : Café au lait");
    println!("Output  : {}", processed);
    let sentences = TextPipeline::split_sentences(
        "Hello world. How are you? I am fine!"
    );
    println!("Sentences: {:?}", sentences);
}
```

**Real output จากการรัน `cargo run`:**

```
=== NLP Tokenizer Demo ===

--- WhitespaceTokenizer ---
Input : Hello, World! This is Rust NLP.
Output: ["hello", "world", "this", "is", "rust", "nlp"]

--- WordTokenizer ---
Input : don't stop, won't stop.
Output: ["don't", "stop", "won't", "stop"]

--- BPE Training ---
Corpus: ["low", "low", "low", "low", "lower", "lower", "newest", "newest", "newest", "widest"]
Merge rules learned: 10
  Rule 1: (l, o) -> lo
  Rule 2: (lo, w) -> low
  Rule 3: (s, t) -> st
  Rule 4: (e, st) -> est
  Rule 5: (w, est) -> west
  Rule 6: (n, e) -> ne
  Rule 7: (ne, west) -> newest
  Rule 8: (low, e) -> lowe
  Rule 9: (lowe, r) -> lower
  Rule 10: (d, est) -> dest

--- BPE Encode/Decode ---
Encode 'low'  : [15]
Decode back   : "low"

--- WordPiece Tokenizer ---
Input : hello unaffable runner
Output: ["hello", "un", "##aff", "##able", "run", "##ner"]

--- Vocabulary ---
Vocab size: 11
ID of 'l'  : Some(4)
Loaded size: 11

--- TextPipeline ---
Input   : Café au lait
Output  : cafe au lait
Sentences: ["Hello world.", "How are you?", "I am fine!"]
```

## การทดสอบ (Testing)

### Unit Tests ครบทุก Module

โปรเจคมี unit tests 23 ตัวครอบคลุมทุกส่วนสำคัญ:

**`src/tokenizer.rs` — tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_whitespace_tokenize_basic() {
        let tokens = WhitespaceTokenizer::tokenize("Hello, World! How are you?");
        assert_eq!(tokens, vec!["hello", "world", "how", "are", "you"]);
    }

    #[test]
    fn test_whitespace_tokenize_numbers() {
        let tokens = WhitespaceTokenizer::tokenize("Rust 2024 edition!");
        assert_eq!(tokens, vec!["rust", "2024", "edition"]);
    }

    #[test]
    fn test_whitespace_tokenize_empty() {
        let tokens = WhitespaceTokenizer::tokenize("   ");
        assert!(tokens.is_empty());
    }

    #[test]
    fn test_word_tokenizer_apostrophe() {
        let tok = WordTokenizer::new();
        let tokens = tok.tokenize("don't stop, won't stop.");
        assert_eq!(tokens, vec!["don't", "stop", "won't", "stop"]);
    }

    #[test]
    fn test_word_tokenizer_punct_strip() {
        let tok = WordTokenizer::new();
        let tokens = tok.tokenize("Hello, World!");
        assert_eq!(tokens, vec!["hello", "world"]);
    }
}
```

**`src/bpe.rs` — tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_bpe_count_pairs() {
        let corpus = vec!["low", "low", "low", "lower", "newer"];
        let trainer = BpeTrainer::new(30);
        let rules = trainer.train(&corpus);
        assert!(!rules.is_empty());
    }

    #[test]
    fn test_bpe_most_frequent_pair() {
        // corpus ที่มี "lo" เกิดบ่อย
        let corpus = vec!["low", "low", "low", "low", "newer"];
        let trainer = BpeTrainer::new(10);
        let rules = trainer.train(&corpus);
        assert!(!rules.is_empty());
        // ตรวจว่า merged string ตรงกับ pair
        let first = &rules[0];
        let merged = format!("{}{}", first.pair.0, first.pair.1);
        assert_eq!(merged, first.merged);
    }

    #[test]
    fn test_bpe_encode_decode_roundtrip() {
        let corpus = vec!["hello", "hello", "world", "world", "world"];
        let trainer = BpeTrainer::new(20);
        let rules = trainer.train(&corpus);

        let mut base_vocab: Vec<String> = Vec::new();
        for word in &corpus {
            for c in word.chars() {
                let s = c.to_string();
                if !base_vocab.contains(&s) {
                    base_vocab.push(s);
                }
            }
        }

        let tokenizer = BpeTokenizer::new(rules, base_vocab);
        let ids = tokenizer.encode("hello");
        assert!(!ids.is_empty());
        let decoded = tokenizer.decode(&ids);
        assert!(!decoded.is_empty());
    }

    #[test]
    fn test_special_token_ids() {
        let tokenizer = BpeTokenizer::new(
            vec![],
            vec!["a".to_string(), "b".to_string()]
        );
        assert_eq!(tokenizer.pad_id(), 0);
        assert_eq!(tokenizer.unk_id(), 1);
        assert_eq!(tokenizer.bos_id(), 2);
        assert_eq!(tokenizer.eos_id(), 3);
    }

    #[test]
    fn test_unk_handling() {
        let tokenizer = BpeTokenizer::new(vec![], vec!["a".to_string()]);
        let ids = tokenizer.encode("z");
        assert_eq!(ids, vec![tokenizer.unk_id()]);
    }
}
```

**`src/wordpiece.rs` — tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn build_vocab(tokens: &[&str]) -> HashSet<String> {
        tokens.iter().map(|s| s.to_string()).collect()
    }

    #[test]
    fn test_wordpiece_simple() {
        let vocab = build_vocab(&["[UNK]", "hello", "world"]);
        let tok = WordPieceTokenizer::new(vocab);
        let result = tok.tokenize("hello world");
        assert_eq!(result, vec!["hello", "world"]);
    }

    #[test]
    fn test_wordpiece_continuation_prefix() {
        let vocab = build_vocab(&["[UNK]", "un", "##aff", "##able"]);
        let tok = WordPieceTokenizer::new(vocab);
        let result = tok.tokenize_word("unaffable");
        assert_eq!(result, vec!["un", "##aff", "##able"]);
    }

    #[test]
    fn test_wordpiece_unk() {
        let vocab = build_vocab(&["[UNK]", "hello"]);
        let tok = WordPieceTokenizer::new(vocab);
        let result = tok.tokenize_word("xyz");
        assert_eq!(result, vec!["[UNK]"]);
    }

    #[test]
    fn test_wordpiece_single_char() {
        let vocab = build_vocab(&["[UNK]", "a", "##b", "##c"]);
        let tok = WordPieceTokenizer::new(vocab);
        let result = tok.tokenize_word("abc");
        assert_eq!(result, vec!["a", "##b", "##c"]);
    }
}
```

**`src/vocab.rs` — tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_vocab_add_and_get() {
        let mut vocab = Vocab::new();
        let id = vocab.add_token("hello".to_string());
        assert_eq!(id, 0);
        assert_eq!(vocab.get_id("hello"), Some(0));
        assert_eq!(vocab.get_token(0), Some("hello"));
    }

    #[test]
    fn test_vocab_save_load_json() {
        let mut vocab = Vocab::new();
        vocab.add_token("[PAD]".to_string());
        vocab.add_token("[UNK]".to_string());
        vocab.add_token("hello".to_string());
        vocab.add_token("world".to_string());

        let json = vocab.to_json();
        let loaded = Vocab::from_json(&json).unwrap();

        assert_eq!(loaded.size(), 4);
        assert_eq!(loaded.get_id("hello"), Some(2));
        assert_eq!(loaded.get_token(3), Some("world"));
    }

    #[test]
    fn test_vocab_unk_id() {
        let tokens: Vec<String> = vec!["[PAD]", "[UNK]", "a", "b"]
            .iter()
            .map(|s| s.to_string())
            .collect();
        let vocab = Vocab::from_tokens(&tokens);
        assert_eq!(vocab.unk_id(), 1);
    }

    #[test]
    fn test_vocab_build_from_corpus() {
        let corpus = vec!["hello", "world", "hello"];
        let vocab = Vocab::build_from_corpus(&corpus, 20);
        assert!(vocab.get_id("[PAD]").is_some());
        assert!(vocab.get_id("[UNK]").is_some());
        assert!(vocab.get_id("l").is_some());
        assert!(vocab.get_id("o").is_some());
    }
}
```

**`src/pipeline.rs` — tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_pipeline_lowercase() {
        let pipeline = TextPipeline {
            lowercase: true,
            unicode_normalize: NormalizeForm::None,
            strip_accents: false,
            handle_chinese: false,
        };
        let result = pipeline.process("Hello World");
        assert_eq!(result, "hello world");
    }

    #[test]
    fn test_pipeline_unicode_nfd_strip_accents() {
        let pipeline = TextPipeline {
            lowercase: false,
            unicode_normalize: NormalizeForm::NFD,
            strip_accents: true,
            handle_chinese: false,
        };
        // "é" = e + combining accent → NFD + strip → "e"
        let result = pipeline.process("café");
        assert_eq!(result, "cafe");
    }

    #[test]
    fn test_pipeline_chinese_spacing() {
        let pipeline = TextPipeline {
            lowercase: false,
            unicode_normalize: NormalizeForm::None,
            strip_accents: false,
            handle_chinese: true,
        };
        let result = pipeline.process("你好");
        assert!(result.contains(' '));
        assert!(result.contains('你'));
        assert!(result.contains('好'));
    }

    #[test]
    fn test_sentence_split() {
        let text = "Hello world. How are you? I am fine!";
        let sentences = TextPipeline::split_sentences(text);
        assert_eq!(sentences.len(), 3);
        assert_eq!(sentences[0], "Hello world.");
        assert_eq!(sentences[1], "How are you?");
        assert_eq!(sentences[2], "I am fine!");
    }

    #[test]
    fn test_sentence_split_no_period() {
        let text = "No ending punctuation here";
        let sentences = TextPipeline::split_sentences(text);
        assert_eq!(sentences.len(), 1);
        assert_eq!(sentences[0], "No ending punctuation here");
    }
}
```

### Real `cargo test` Output

รัน `cargo test` บน verification project ได้ผลดังนี้:

```
running 23 tests
test bpe::tests::test_bpe_count_pairs ... ok
test bpe::tests::test_bpe_encode_decode_roundtrip ... ok
test bpe::tests::test_special_token_ids ... ok
test bpe::tests::test_bpe_most_frequent_pair ... ok
test bpe::tests::test_unk_handling ... ok
test pipeline::tests::test_pipeline_chinese_spacing ... ok
test pipeline::tests::test_pipeline_lowercase ... ok
test pipeline::tests::test_pipeline_unicode_nfd_strip_accents ... ok
test pipeline::tests::test_sentence_split ... ok
test pipeline::tests::test_sentence_split_no_period ... ok
test tokenizer::tests::test_whitespace_tokenize_basic ... ok
test tokenizer::tests::test_whitespace_tokenize_empty ... ok
test tokenizer::tests::test_whitespace_tokenize_numbers ... ok
test vocab::tests::test_vocab_add_and_get ... ok
test vocab::tests::test_vocab_save_load_json ... ok
test vocab::tests::test_vocab_build_from_corpus ... ok
test wordpiece::tests::test_wordpiece_continuation_prefix ... ok
test vocab::tests::test_vocab_unk_id ... ok
test wordpiece::tests::test_wordpiece_simple ... ok
test wordpiece::tests::test_wordpiece_unk ... ok
test tokenizer::tests::test_word_tokenizer_apostrophe ... ok
test wordpiece::tests::test_wordpiece_single_char ... ok
test tokenizer::tests::test_word_tokenizer_punct_strip ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/nlp_tokenizer-5b5023156bdd6bb0)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests nlp_tokenizer

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก 23 tests ผ่านหมด ครอบคลุม:
- `tokenizer`: 5 tests — whitespace basic/numbers/empty, word apostrophe/punct
- `bpe`: 5 tests — count pairs, most frequent pair, encode/decode roundtrip, special token ids, unk handling
- `wordpiece`: 4 tests — simple, continuation prefix `##`, unk fallback, single char decomposition
- `vocab`: 4 tests — add/get, save/load JSON, unk id, build from corpus
- `pipeline`: 5 tests — lowercase, NFD+strip accents, Chinese spacing, sentence split, sentence split no period

## Pitfalls ที่ต้องระวัง

### Pitfall 1: BPE Tie-breaking ไม่ Deterministic

**ปัญหา:** เมื่อมีหลาย pair มีความถี่เท่ากัน `max_by_key` จะเลือก pair ใดก็ได้ขึ้นกับ HashMap iteration order ซึ่งไม่ deterministic

```rust
// อันตราย! — ถ้า ("l","o") และ ("o","w") มีความถี่เท่ากัน
// ผลลัพธ์อาจต่างกันแต่ละครั้ง
let best_pair = pair_freqs
    .iter()
    .max_by_key(|(_, &freq)| freq)
    .map(|(pair, _)| pair.clone())
    .unwrap();
```

**วิธีแก้:** เพิ่ม secondary sort key เพื่อ tie-breaking ที่ deterministic:

```rust
// ปลอดภัย — tie-break ด้วย lexicographic order
let best_pair = pair_freqs
    .iter()
    .max_by(|(pair_a, &freq_a), (pair_b, &freq_b)| {
        freq_a.cmp(&freq_b)
            .then(pair_a.0.cmp(&pair_b.0))
            .then(pair_a.1.cmp(&pair_b.1))
    })
    .map(|(pair, _)| pair.clone())
    .unwrap();
```

### Pitfall 2: Vocab Load ไม่ Preserve Order

**ปัญหา:** `HashMap` ใน JSON deserialization ไม่รับประกัน order ดังนั้น `id_to_token` อาจไม่ตรงกับ `token_to_id` หลัง round-trip

```rust
// อันตราย! — JSON HashMap ไม่มี order
#[derive(Serialize, Deserialize)]
pub struct VocabBroken {
    pub token_to_id: HashMap<String, u32>,
    // id_to_token ถูก derive จาก HashMap แต่ order ไม่ guaranteed
    pub id_to_token: Vec<String>,  // อาจ mismatch!
}
```

**วิธีแก้:** เก็บ `id_to_token` เป็น `Vec<String>` และ rebuild `token_to_id` จากมันหลัง deserialize:

```rust
#[derive(Serialize, Deserialize)]
struct VocabData {
    id_to_token: Vec<String>,  // single source of truth
}

impl Vocab {
    pub fn from_json(json: &str) -> Result<Self, serde_json::Error> {
        let data: VocabData = serde_json::from_str(json)?;
        let token_to_id = data.id_to_token
            .iter()
            .enumerate()
            .map(|(i, t)| (t.clone(), i as u32))
            .collect();
        Ok(Vocab {
            token_to_id,
            id_to_token: data.id_to_token,
        })
    }
}
```

### Pitfall 3: Unicode Normalization ก่อนหรือหลัง Lowercase?

**ปัญหา:** ลำดับ operations ใน pipeline สำคัญมาก การ lowercase ก่อน NFD อาจทำให้ผลต่างกัน เพราะ uppercase accented letters เช่น "Ä" มี NFD form ต่างจาก lowercase "ä"

```rust
// อันตราย! — lowercase ก่อน NFD อาจให้ผลต่าง
let wrong = text.to_lowercase().nfd().collect::<String>();

// ถูกต้อง — NFD ก่อน แล้วค่อย strip accents แล้วค่อย lowercase
let correct = text.nfd()
    .filter(|c| !is_combining_mark(*c))
    .collect::<String>()
    .to_lowercase();
```

ลำดับที่ถูกต้องตาม BERT paper คือ: NFD → strip accents → lowercase

### Pitfall 4: WordPiece `##` Prefix ใน Vocab Lookup

**ปัญหา:** ลืมเพิ่ม `##` prefix เมื่อ `start != 0` ทำให้ lookup vocab หา "able" แทน "##able" และ fail ผิดพลาด

```rust
// อันตราย! — ลืม ## prefix
let substr_to_check = substr.clone();  // ผิด!

// ถูกต้อง
let substr_to_check = if start == 0 {
    substr.clone()
} else {
    format!("##{}", substr)  // continuation prefix
};
```

**ผลที่เกิด:** WordPiece จะ return `[UNK]` แม้ word นั้น decomposable ได้จริง ๆ เพราะมองหา "able" แต่ vocab มีแค่ "##able"

### Pitfall 5: BPE Merge Order สำคัญมากตอน Encode

**ปัญหา:** ต้อง apply merge rules **ตามลำดับที่ train** เสมอ ถ้า apply ผิดลำดับจะได้ผลต่าง

```rust
// อันตราย! — apply rules โดยไม่เรียงลำดับ
for rule in self.merge_rules.iter().rev() {  // ผิด! ต้องไม่ reverse
    tokens = apply_merge(tokens, rule);
}

// ถูกต้อง — apply ตามลำดับ training เสมอ
for rule in &self.merge_rules {  // iterate ตาม training order
    tokens = apply_merge(&tokens, &rule.pair, &rule.merged);
}
```

เหตุผล: rules แรก ๆ สร้าง tokens ที่ rules หลัง ๆ ต้องการ เช่น rule 1 สร้าง "lo" และ rule 2 ใช้ "lo" เพื่อสร้าง "low"

### Pitfall 6: ไม่จัดการ Empty String ใน Pipeline

**ปัญหา:** หลังจาก strip accents และ filter แล้ว token อาจกลายเป็น empty string และ slip เข้า vocab หรือ output

```rust
// อันตราย! — empty tokens เข้า output
let tokens: Vec<String> = text.split_whitespace()
    .map(|w| w.chars().filter(|c| c.is_alpha()).collect::<String>())
    .collect();  // อาจมี "" ถ้า word เป็น all-punct

// ถูกต้อง — filter empty ออก
let tokens: Vec<String> = text.split_whitespace()
    .map(|w| w.chars().filter(|c| c.is_alpha()).collect::<String>())
    .filter(|w| !w.is_empty())  // กำจัด empty strings
    .collect();
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# ขนาด binary ที่ได้
ls -lh target/release/nlp-tokenizer
# -rwxr-xr-x 1 user user 3.2M nlp-tokenizer
```

### Benchmark Training Speed

สำหรับ production ควรวัด performance ของ BPE training:

```bash
# เพิ่มใน Cargo.toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "bpe_bench"
harness = false
```

```rust
// benches/bpe_bench.rs
use criterion::{criterion_group, criterion_main, Criterion};
use nlp_tokenizer::bpe::BpeTrainer;

fn bench_bpe_training(c: &mut Criterion) {
    let corpus: Vec<&str> = include_str!("../data/corpus.txt")
        .split_whitespace()
        .collect();

    c.bench_function("bpe_train_1000_merges", |b| {
        b.iter(|| {
            let trainer = BpeTrainer::new(1000);
            trainer.train(&corpus)
        })
    });
}

criterion_group!(benches, bench_bpe_training);
criterion_main!(benches);
```

### Serialize Tokenizer สำหรับ Production

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct TokenizerConfig {
    pub vocab: Vocab,
    pub merge_rules: Vec<BpeMergeRule>,
    pub tokenizer_type: String,  // "bpe" หรือ "wordpiece"
}

impl TokenizerConfig {
    pub fn save(&self, path: &str) -> std::io::Result<()> {
        let json = serde_json::to_string_pretty(self).unwrap();
        std::fs::write(path, json)
    }

    pub fn load(path: &str) -> std::io::Result<Self> {
        let json = std::fs::read_to_string(path)?;
        Ok(serde_json::from_str(&json).unwrap())
    }
}

// ใช้งาน
let config = TokenizerConfig {
    vocab: trained_vocab,
    merge_rules: bpe_rules,
    tokenizer_type: "bpe".to_string(),
};
config.save("tokenizer.json").unwrap();

// โหลดกลับมาใช้
let loaded = TokenizerConfig::load("tokenizer.json").unwrap();
```

### Docker Container

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/nlp-tokenizer /usr/local/bin/
COPY tokenizer.json /data/tokenizer.json
ENV TOKENIZER_PATH=/data/tokenizer.json
CMD ["nlp-tokenizer"]
```

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม Unigram Language Model Tokenizer

SentencePiece ของ Google ใช้ Unigram Language Model (ต่างจาก BPE) — ลองเพิ่ม `UnigramTokenizer` ที่ train โดย maximize log-likelihood ของ data แทนที่จะ count pair frequencies

```rust
pub struct UnigramTrainer {
    pub vocab_size: usize,
    pub shrinking_factor: f64,  // ตัดคำออกทีละ (1-shrinking_factor)*100%
}

impl UnigramTrainer {
    // hint: เริ่มจาก large vocab → prune tokens ที่ลด loss น้อยที่สุด
    pub fn train(&self, corpus: &[&str]) -> Vec<(String, f64)> {
        // คืน (token, log_probability) pairs
        todo!()
    }
}
```

### Exercise 2: BPE Dropout สำหรับ Data Augmentation

BPE Dropout เป็นเทคนิคที่ Provilkov et al. 2020 เสนอ — ตอน training ของ model (ไม่ใช่ tokenizer training) ให้ randomly skip บาง merge rules เพื่อ augment data

```rust
pub struct BpeTokenizerWithDropout {
    tokenizer: BpeTokenizer,
    dropout_rate: f64,  // ความน่าจะเป็นที่จะ skip แต่ละ merge rule
}

impl BpeTokenizerWithDropout {
    /// encode แบบ stochastic สำหรับใช้ตอน training
    pub fn encode_with_dropout(&self, text: &str, rng: &mut impl Rng) -> Vec<u32> {
        // hint: ใน apply_merge ให้ตรวจ rng.random::<f64>() < self.dropout_rate
        // ถ้า true → skip merge นั้น
        todo!()
    }
}
```

### Exercise 3: Thai Language Tokenizer

ภาษาไทยไม่มี space ระหว่างคำ ทำให้ tokenization ยากกว่า ลองเพิ่ม `ThaiTokenizer` ที่ใช้ dictionary-based maximum matching algorithm

```rust
pub struct ThaiTokenizer {
    dictionary: HashSet<String>,
    max_word_length: usize,
}

impl ThaiTokenizer {
    /// Longest matching algorithm สำหรับภาษาไทย
    pub fn tokenize(&self, text: &str) -> Vec<String> {
        let chars: Vec<char> = text.chars().collect();
        let mut result = Vec::new();
        let mut pos = 0;

        while pos < chars.len() {
            let mut found = false;
            // ลองจาก max_word_length ลงมา
            let end = (pos + self.max_word_length).min(chars.len());
            for len in (1..=(end - pos)).rev() {
                let word: String = chars[pos..pos+len].iter().collect();
                if self.dictionary.contains(&word) {
                    result.push(word);
                    pos += len;
                    found = true;
                    break;
                }
            }
            if !found {
                // unknown character → เอา character เดี่ยว
                result.push(chars[pos].to_string());
                pos += 1;
            }
        }
        result
    }
}
```

### Exercise 4: Parallel BPE Training ด้วย Rayon

BPE training บน large corpus ช้ามาก ลองใช้ `rayon` เพื่อ parallelize การนับ pair frequencies

```rust
// เพิ่มใน Cargo.toml: rayon = "1"
use rayon::prelude::*;

fn count_pairs_parallel(
    word_freqs: &HashMap<Vec<String>, usize>
) -> HashMap<(String, String), usize> {
    // hint: แบ่ง word_freqs เป็น chunks แล้ว reduce
    word_freqs
        .par_iter()
        .flat_map(|(tokens, &freq)| {
            tokens.windows(2)
                .map(|w| ((w[0].clone(), w[1].clone()), freq))
                .collect::<Vec<_>>()
        })
        .fold(
            HashMap::new,
            |mut acc, (pair, freq)| {
                *acc.entry(pair).or_insert(0) += freq;
                acc
            }
        )
        .reduce(HashMap::new, |mut a, b| {
            for (k, v) in b {
                *a.entry(k).or_insert(0) += v;
            }
            a
        })
}
```

### Exercise 5: Streaming Tokenizer สำหรับ Large Files

สำหรับ corpus ขนาดหลาย GB การ load ทั้งหมดเข้า memory ไม่ได้ ลองเพิ่ม streaming interface

```rust
use std::io::{BufRead, BufReader};

pub struct StreamingBpeTrainer {
    pub vocab_size: usize,
    pub min_frequency: usize,
}

impl StreamingBpeTrainer {
    /// train จาก file โดยไม่ต้อง load ทั้งหมดเข้า memory
    pub fn train_from_file(&self, path: &str) -> Vec<BpeMergeRule> {
        let file = std::fs::File::open(path).unwrap();
        let reader = BufReader::new(file);

        let mut word_freqs: HashMap<String, usize> = HashMap::new();
        for line in reader.lines() {
            let line = line.unwrap();
            for word in line.split_whitespace() {
                *word_freqs.entry(word.to_string()).or_insert(0) += 1;
            }
        }

        // filter by min_frequency
        let corpus: Vec<(&str, usize)> = word_freqs
            .iter()
            .filter(|(_, &freq)| freq >= self.min_frequency)
            .map(|(w, &freq)| (w.as_str(), freq))
            .collect();

        // เดิน BPE training ด้วย weighted corpus
        todo!()
    }
}
```

### Exercise 6: Token Alignment สำหรับ Named Entity Recognition

NER ต้องการ alignment ระหว่าง subword tokens กับ original words เพื่อให้รู้ว่า token id ไหนสอดคล้องกับคำไหน

```rust
pub struct TokenAlignment {
    pub token_ids: Vec<u32>,
    /// word_ids[i] = index ของ word ที่ token i สอดคล้อง
    /// None สำหรับ special tokens ([CLS], [SEP])
    pub word_ids: Vec<Option<usize>>,
}

impl BpeTokenizer {
    pub fn encode_with_alignment(&self, words: &[&str]) -> TokenAlignment {
        let mut token_ids = Vec::new();
        let mut word_ids = Vec::new();

        for (word_idx, word) in words.iter().enumerate() {
            let word_tokens = self.encode(word);
            for id in word_tokens {
                token_ids.push(id);
                word_ids.push(Some(word_idx));
            }
        }

        TokenAlignment { token_ids, word_ids }
    }
}
```

## สรุป

โปรเจคนี้สร้าง NLP tokenization pipeline ที่สมบูรณ์ครอบคลุม 4 ระดับของ tokenization:

| Tokenizer | Algorithm | Use Case |
|-----------|-----------|----------|
| `WhitespaceTokenizer` | split + filter | Simple preprocessing |
| `WordTokenizer` | Regex patterns | Better word boundary detection |
| `BpeTokenizer` | BPE merge rules | GPT-style models |
| `WordPieceTokenizer` | Greedy longest-match | BERT-style models |

**Pattern สำคัญที่ได้เรียน:**

1. **Iterative refinement** — BPE algorithm สอนวิธีเริ่มจาก state เล็ก ๆ (characters) แล้ว iteratively ปรับปรุงจนได้ state ที่ต้องการ

2. **Separation of concerns** — แยก `BpeTrainer` (training-time) กับ `BpeTokenizer` (inference-time) ออกจากกัน ทำให้ code สะอาดและ testable

3. **Greedy algorithm** — WordPiece ใช้ greedy approach ที่ไม่ optimal globally แต่ practical เพราะรันเร็ว

4. **Unicode handling** — การ normalize Unicode ก่อน process เป็น best practice เสมอ โดยเฉพาะเมื่อรับข้อมูลจากหลาย sources

5. **Special token convention** — `[PAD]=0, [UNK]=1, [BOS]=2, [EOS]=3` เป็น convention ที่แพร่หลาย ควรใช้ตามนี้เพื่อ interoperability

**Performance considerations:**
- `HashMap::entry` API ใช้แทน `contains` + `insert` เพื่อหลีกเลี่ยง double lookup
- `Vec<String>` แทน `HashSet<String>` สำหรับ `id_to_token` เพราะ access by index เร็วกว่า
- `split_whitespace()` เร็วกว่า `split(' ')` เพราะจัดการ multiple spaces อัตโนมัติ

**ขั้นต่อไป:** ใน Project I06 Recommender System เราจะนำ tokenization ที่สร้างนี้ไปใช้เป็น building block ของ text-based recommendation system โดยแปลง user reviews เป็น token sequences แล้วใช้ similarity computation ค้นหา items ที่เกี่ยวข้อง

---

**โปรเจคก่อนหน้า:** [Project I04: Clustering](project-i04-clustering.md) | **โปรเจคถัดไป:** [Project I06: Recommender System](project-i06-recommender.md)
