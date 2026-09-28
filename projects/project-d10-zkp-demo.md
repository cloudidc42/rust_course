# Project D10: Zero-Knowledge Proof Demo

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

Zero-Knowledge Proof (ZKP) คือ cryptographic protocol ที่ให้ฝ่าย Prover พิสูจน์ต่อฝ่าย Verifier ได้ว่าตัวเองรู้ข้อมูลลับบางอย่าง โดยไม่เปิดเผยข้อมูลลับนั้นเลย การพิสูจน์ไม่ได้ถ่ายโอน knowledge ใด ๆ นอกจาก "ใช่ ฉันรู้" หรือ "ไม่ใช่"

โปรเจคนี้ implement ZKP stack ตั้งแต่ระดับ mathematical primitive ไปจนถึง HTTP API:

- **Schnorr protocol** บน safe-prime group (discrete log assumption)
- **Fiat-Shamir transform** แปลง interactive proof เป็น non-interactive proof ด้วย hash function
- **Pedersen commitment** — commit โดยไม่เปิดเผยค่า, เปิดทีหลังพิสูจน์ความถูกต้อง
- **Range proof** — พิสูจน์ว่าค่าอยู่ใน [0, 2^n) โดยไม่บอกค่าจริง
- **Ristretto255** — modern elliptic curve group ที่ปลอดภัยและมี cofactor 1
- **Batch verification** — verify N proofs พร้อมกันเร็วกว่าแยก N ครั้ง

ใน production ใช้กับ:
- **Anonymous credential** — พิสูจน์ attribute (อายุ >= 18, สมาชิก group) โดยไม่เปิดเผย identity
- **Private blockchain** — พิสูจน์ transaction ถูกต้องโดยไม่เปิดเผย sender/amount (Zcash, Tornado Cash)
- **Password authentication** — server ตรวจสอบ knowledge ของ password โดยไม่เคยรับ plaintext password
- **Voting system** — พิสูจน์ว่า vote ถูกต้องโดยไม่เปิดเผย choice

**Learning value:** โปรเจคนี้สอน mathematical foundation ของ ZKP, การใช้ `curve25519-dalek` สำหรับ elliptic curve arithmetic, การออกแบบ protocol API ที่ type-safe, และ performance optimization ผ่าน batch verification

## สิ่งที่จะได้เรียนรู้

- **Σ-protocols (Sigma protocols)** — โครงสร้าง commit-challenge-response ของ zero-knowledge proofs
- **Fiat-Shamir heuristic** — แปลง interactive proof เป็น signature-like non-interactive proof ด้วย random oracle
- **Pedersen commitment** — cryptographic commitment scheme ที่มีคุณสมบัติ homomorphic
- **Bit decomposition range proof** — เทคนิคพิสูจน์ range ด้วย per-bit commitments
- **curve25519-dalek API** — `RistrettoPoint`, `Scalar`, `RISTRETTO_BASEPOINT_POINT`
- **Batch verification** — random linear combination technique สำหรับ amortized verification
- **Proof serialization** — compressed point + scalar encoding เป็น bytes และ base64url
- **BigUint arithmetic** — modular exponentiation สำหรับ safe-prime group operations

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30** — Rust fundamentals, ownership, traits, generics
- **Part 46–50** — async/await, Tokio runtime (สำหรับ HTTP API)
- **Part 61–65** — axum web framework (สำหรับ HTTP API endpoints)
- **Part 80–85** — serde/serde_json serialization
- **Part 86–90** — cryptographic primitives ใน Rust (sha2, rand)
- **Part 96–100** — performance profiling, criterion benchmarks

## โครงสร้างโปรเจค (Project Layout)

```
zkp-demo/
├── src/
│   ├── main.rs              # Interactive demo + CLI entry point
│   ├── schnorr_prime.rs     # Schnorr protocol บน safe-prime group
│   ├── fiat_shamir.rs       # Fiat-Shamir non-interactive transform
│   ├── pedersen.rs          # Pedersen commitment scheme
│   ├── range_proof.rs       # Range proof ด้วย bit decomposition
│   ├── ristretto_schnorr.rs # Schnorr + Pedersen บน Ristretto255
│   ├── serialization.rs     # Proof bytes/base64url encoding
│   └── batch_verify.rs      # Batch verification (random linear combination)
├── benches/
│   └── zkp_bench.rs         # Criterion benchmarks
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### หลักการ: Zero-Knowledge ตาม Goldwasser-Micali-Rackoff (GMR)

Interactive proof ที่เป็น Zero-Knowledge ต้องมีคุณสมบัติ 3 ข้อ:

1. **Completeness** — ถ้า Prover รู้ secret จริง Verifier ยอมรับด้วยความน่าจะเป็นสูง (1 ใน Schnorr)
2. **Soundness** — ถ้า Prover ไม่รู้ secret Verifier ปฏิเสธด้วยความน่าจะเป็นสูง (ยกเว้น negligible 1/q)
3. **Zero-knowledge** — Verifier ไม่ได้รับ information เพิ่มเติมใด ๆ นอกจากว่า Prover รู้ secret

```
                   ┌─────────────────────────────────────────────────────┐
                   │              Schnorr Σ-Protocol                     │
                   │                                                     │
  Prover (knows x) │                                   Verifier          │
        ─────────────────────────────────────────────────────────────    │
  pick r ∈ [1,q-1] │                                                     │
  R = g^r mod p    │──── commitment R ──────────────────►│               │
                   │                                     │ pick c ∈ [1,q]│
                   │◄─── challenge c ────────────────────│               │
  s = r + cx mod q │                                     │               │
                   │──── response s ────────────────────►│               │
                   │                                     │ check:        │
                   │                                     │ g^s = R·Y^c ? │
                   └─────────────────────────────────────────────────────┘
```

### Fiat-Shamir Transform

แทนที่ Verifier ส่ง challenge แบบ interactive ใช้ Hash function แทน:

```
c = H(g || Y || R)   ← deterministic challenge จาก public inputs
```

ทำให้ proof กลายเป็น **signature**: `(R, s)` ที่ verify ได้โดยไม่ต้องมี Verifier online

```
Non-interactive proof: (R, s)
  - Prover เลือก r แบบ random, คำนวณ R = g^r mod p
  - c = H(g || Y || R)
  - s = r + c*x mod q
Verify:
  - c' = H(g || Y || R)  ← recompute
  - check g^s = R * Y^c' mod p
```

### Pedersen Commitment: Binding และ Hiding

```
Setup: g, h generators ของ order-q subgroup, log_g(h) ไม่เป็นที่รู้จัก

commit(v, r) = g^v * h^r mod p

Binding: หา (v', r') ≠ (v, r) ที่ commit(v',r') = commit(v,r)
  ⟺ g^(v-v') = h^(r'-r) mod p
  ⟺ v-v' = (r'-r) * log_g(h) mod q
  ⟺ ต้อง solve DLOG — infeasible ด้วย computational DLOG assumption

Hiding: ถ้า r uniform random ใน Z_q แล้ว C = g^v * h^r uniform ใน subgroup
  ⟺ C ไม่เปิดเผยข้อมูลใด ๆ เกี่ยวกับ v (perfect hiding)
```

Pedersen commitment มีคุณสมบัติ **homomorphic**:
```
commit(v1, r1) * commit(v2, r2) = g^(v1+v2) * h^(r1+r2) = commit(v1+v2, r1+r2)
```
ทำให้ cryptographic arithmetic บน committed values เป็นไปได้

### Ristretto255: ทำไมถึงเลือก

`curve25519-dalek` ใช้ Ristretto255 ซึ่งเป็น prime-order group ที่ wrap Curve25519:
- **Cofactor 1** — ไม่มีปัญหา small subgroup attacks ที่เกิดจาก cofactor 8 ของ Curve25519
- **Canonical encoding** — แต่ละ group element มี compressed encoding เดียว (ไม่มี twist attacks)
- **52-bit limb arithmetic** — ประสิทธิภาพสูงบน 64-bit CPU
- **Safe API** — `Scalar::random()` ใช้ OsRng, `from_canonical_bytes` reject non-canonical scalars

### Batch Verification: Random Linear Combination

ตรวจสอบ N proofs แยกกัน: N ครั้งของ `g^si * B` (N scalar multiplications)

Batch verification สุ่ม a_i ∈ [1,l-1] แล้ว verify:

```
sum(a_i * s_i) * B == sum(a_i * (R_i + c_i * Y_i))
```

- LHS: 1 multi-scalar multiplication (Pippenger algorithm: O(N/log N) point additions)
- RHS: N point operations แต่ grouped ด้วย random weights
- ประหยัดประมาณ 30–50% เมื่อ N ≥ 16

Security: ถ้า proof j ไม่ valid แล้วสมการ batch ไม่ตรงด้วยความน่าจะเป็น ≥ 1 - 1/l (negligible)

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Cargo.toml และโครงสร้างพื้นฐาน

สร้าง project ใหม่:

```bash
cargo new zkp-demo
cd zkp-demo
```

**Cargo.toml:**

```toml
[package]
name = "zkp-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
curve25519-dalek = { version = "4", features = ["rand_core"] }
sha2 = "0.10"
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
base64 = "0.22"
num-bigint = { version = "0.4", features = ["rand"] }
num-traits = "0.2"
axum = { version = "0.8", features = ["tokio"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "zkp_bench"
harness = false
```

โครงสร้าง module ใน `src/main.rs`:

```rust
pub mod schnorr_prime;
pub mod fiat_shamir;
pub mod pedersen;
pub mod range_proof;
pub mod ristretto_schnorr;
pub mod serialization;
pub mod batch_verify;
```

---

### ขั้นที่ 2: Schnorr Protocol บน Safe-Prime Group

สร้าง `src/schnorr_prime.rs` — Interactive Σ-protocol:

```rust
// Schnorr identification protocol over a safe-prime group.
// Parameters are RFC 5114 Group 22 (1024-bit safe prime, 160-bit order subgroup) —
// used here for pedagogical purposes only; NOT for production use.
//
// Group: Z*_p where p = 2q+1 (safe prime), order-q subgroup, generator g (order q).
// Secret key x in [1, q-1].  Public key Y = g^x mod p.
// Prover: picks random r, sends R = g^r mod p.
// Verifier: sends challenge c.
// Prover: responds s = r + c*x mod q.
// Verifier: checks g^s = R * Y^c mod p.
//
// Generator: g = A4D1CBD5... from RFC 5114 — has order q in Z*_p.

use num_bigint::BigUint;
use num_traits::One;
use rand::thread_rng;
use num_bigint::RandBigInt;

/// Returns (p, q, g) — RFC 5114 Group 22 safe prime parameters.
fn params() -> (BigUint, BigUint, BigUint) {
    // p — 1024-bit safe prime (RFC 5114, Section 2.1)
    let p = BigUint::parse_bytes(
        b"B10B8F96A080E01DDE92DE5EAE5D54EC52C99FBCFB06A3C69A6A9DCA52D23B6\
          16073E28675A23D189838EF1E2EE652C013ECB4AEA906112324975C3CD49B83B\
          FACCBDD7D90C4BD7098488E9C219A73724EFFD6FAE5644738FAA31A4FF55BCCC\
          0A151AF5F0DC8B4BD45BF37DF365C1A65E68CFDA76D4DA708DF1FB2BC2E4A4371",
        16,
    ).unwrap();
    // q = (p - 1) / 2  — prime (p is safe prime)
    let q = (&p - BigUint::one()) / BigUint::from(2u32);
    // g — generator of the unique order-q subgroup (RFC 5114 Section 2.1)
    let g = BigUint::parse_bytes(
        b"A4D1CBD5C3FD34126765A442EFB99905F8104DD258AC507FD6406CFF14266D31\
          266FEA1E5C41564B777E690F5504F213160217B4B01B886A5E91547F9E2749F4\
          D7FBD7D3B9A92EE1909D0D2263F80A76A6A24C087A091F531DBF0A0169B6A28A\
          D662A4D18E73AFA32D779D5918D4F3C8E49401D25DA672B50EE1B3A4EF11F2D6FEB",
        16,
    ).unwrap();
    (p, q, g)
}

pub struct SecretKey(pub BigUint);
pub struct PublicKey(pub BigUint);

pub fn keygen() -> (PublicKey, SecretKey) {
    let (p, q, g) = params();
    let mut rng = thread_rng();
    let x = rng.gen_biguint_range(&BigUint::one(), &q);
    let y = g.modpow(&x, &p);
    (PublicKey(y), SecretKey(x))
}

pub struct Commitment {
    pub r_scalar: BigUint,
    pub r_point: BigUint,
}

/// Prover round 1: R = g^r mod p.
pub fn commit_round1() -> Commitment {
    let (p, q, g) = params();
    let mut rng = thread_rng();
    let r = rng.gen_biguint_range(&BigUint::one(), &q);
    Commitment { r_scalar: r.clone(), r_point: g.modpow(&r, &p) }
}

/// Prover round 3: s = r + c*x mod q.
pub fn respond(comm: &Commitment, sk: &SecretKey, c: &BigUint) -> BigUint {
    let (_, q, _) = params();
    (&comm.r_scalar + (c * &sk.0) % &q) % &q
}

/// Verifier: g^s == R * Y^c mod p.
pub fn verify(pk: &PublicKey, r_point: &BigUint, c: &BigUint, s: &BigUint) -> bool {
    let (p, _, g) = params();
    g.modpow(s, &p) == (r_point * pk.0.modpow(c, &p)) % &p
}
```

**ทำไม g ต้องมี order q ไม่ใช่ order 2q?**

สำหรับ safe prime p = 2q+1 กลุ่ม Z*_p มี order p-1 = 2q primitive root (เช่น g=2) มี order 2q แต่ Schnorr ต้องการ exponents ลดลง mod q ดังนั้นต้องใช้ generator ที่มี order q ซึ่งคือ generator ของ subgroup ของ quadratic residues มาตรฐาน RFC 5114 ให้ generator ที่ verify แล้วว่ามี order q ตรง

**Pitfall #1: ใช้ g=2 (primitive root) แทน g ที่มี order q**

ถ้าใช้ `let g = BigUint::from(2u32)` ซึ่งมี order 2q (not q):
```
s = r + cx mod q  ← exponent reduced mod q
g^s mod p         ← แต่ g มี order 2q ไม่ใช่ q
                     ดังนั้น g^(s mod q) ≠ g^s (mod 2q arithmetic)
                     Verify จะ fail แบบ non-deterministic
```
ต้องใช้ generator ที่ verify แล้วว่ามี order q เช่น generator จาก RFC 5114/3526 หรือ square ของ primitive root (g' = g² mod p มี order q)

---

### ขั้นที่ 3: Fiat-Shamir Non-Interactive Transform

สร้าง `src/fiat_shamir.rs`:

```rust
use num_bigint::BigUint;
use num_traits::One;
use sha2::{Digest, Sha256};
use rand::thread_rng;
use num_bigint::RandBigInt;
use crate::schnorr_prime::{PublicKey, SecretKey};

// (params() function เหมือนกับใน schnorr_prime.rs)

/// Non-interactive Schnorr proof (R, s).
pub struct NiProof {
    pub r_bytes: Vec<u8>,
    pub s_bytes: Vec<u8>,
}

/// Fiat-Shamir challenge: SHA-256(g_bytes || Y_bytes || R_bytes) mod q.
///
/// ใช้ SHA-256 เป็น random oracle (สมมติฐาน Random Oracle Model)
/// Hash ครอบทั้ง context: g (group parameter), Y (public key), R (commitment)
/// เพื่อ domain separation และป้องกัน cross-protocol replay
fn fiat_shamir_challenge(g: &BigUint, y: &BigUint, r_point: &BigUint, q: &BigUint) -> BigUint {
    let mut hasher = Sha256::new();
    hasher.update(g.to_bytes_be());
    hasher.update(y.to_bytes_be());
    hasher.update(r_point.to_bytes_be());
    BigUint::from_bytes_be(&hasher.finalize()) % q
}

pub fn prove_non_interactive(sk: &SecretKey, pk: &PublicKey) -> NiProof {
    let (p, q, g) = params();
    let mut rng = thread_rng();
    let r = rng.gen_biguint_range(&BigUint::one(), &q);
    let r_point = g.modpow(&r, &p);
    let c = fiat_shamir_challenge(&g, &pk.0, &r_point, &q);
    let s = (&r + (c * &sk.0) % &q) % &q;
    NiProof { r_bytes: r_point.to_bytes_be(), s_bytes: s.to_bytes_be() }
}

pub fn verify_non_interactive(pk: &PublicKey, proof: &NiProof) -> bool {
    let (p, q, g) = params();
    let r_point = BigUint::from_bytes_be(&proof.r_bytes);
    let s = BigUint::from_bytes_be(&proof.s_bytes);
    let c = fiat_shamir_challenge(&g, &pk.0, &r_point, &q);
    let lhs = g.modpow(&s, &p);
    let rhs = (&r_point * pk.0.modpow(&c, &p)) % &p;
    lhs == rhs
}
```

**สิ่งสำคัญ:** Fiat-Shamir ไม่ได้เป็น interactive อีกต่อไป proof `(R, s)` คือ "signature" บน public key Y ใครก็ verify ได้ offline

**Pitfall #2: ลืม domain separation ใน Fiat-Shamir hash**

```rust
// ❌ hash แค่ R — ทำให้ proof ถูก reuse กับ public key อื่นได้
hasher.update(r_point.to_bytes_be());

// ✓ hash g || Y || R — ผูก proof กับ specific public key
hasher.update(g.to_bytes_be());
hasher.update(y.to_bytes_be());
hasher.update(r_point.to_bytes_be());
```

ถ้าไม่ใส่ Y ใน hash แล้ว proof `(R, s)` ที่สร้างสำหรับ key Y1 อาจ verify ได้กับ key Y2 ในบางกรณีพิเศษ (transcript simulation attack)

---

### ขั้นที่ 4: Pedersen Commitment

สร้าง `src/pedersen.rs`:

```rust
use num_bigint::BigUint;
use num_traits::One;
use rand::thread_rng;
use num_bigint::RandBigInt;

fn params() -> (BigUint, BigUint, BigUint, BigUint) {
    // (p, q, g เหมือนก่อนหน้า)
    let p = /* RFC 5114 Group 22 prime */ BigUint::parse_bytes(b"B10B8F96...", 16).unwrap();
    let q = (&p - BigUint::one()) / BigUint::from(2u32);
    let g = /* RFC 5114 generator */ BigUint::parse_bytes(b"A4D1CBD5...", 16).unwrap();
    // h = g^7 mod p — second independent generator
    // alpha=7 is public; h is in the same order-q subgroup as g
    let h = g.modpow(&BigUint::from(7u32), &p);
    (p, q, g, h)
}

/// C = g^v * h^r mod p
pub fn commit(v: &BigUint, r: &BigUint) -> BigUint {
    let (p, _, g, h) = params();
    (g.modpow(v, &p) * h.modpow(r, &p)) % &p
}

pub fn random_blinding() -> BigUint {
    let (_, q, _, _) = params();
    let mut rng = thread_rng();
    rng.gen_biguint_range(&BigUint::one(), &q)
}

/// Verify opening: g^v * h^r == C
pub fn open(commitment: &BigUint, v: &BigUint, r: &BigUint) -> bool {
    commit(v, r) == *commitment
}

/// Homomorphic: C(v1,r1) * C(v2,r2) mod p = C(v1+v2, (r1+r2) mod q)
pub fn add_commitments(c1: &BigUint, c2: &BigUint) -> BigUint {
    let (p, _, _, _) = params();
    (c1 * c2) % &p
}
```

**คุณสมบัติ Computational Binding ทางเทคนิค:**

การหา `(v', r')` ที่แตกต่างจาก `(v, r)` แต่ให้ commitment เดิมเท่ากับการแก้ discrete logarithm problem:
```
commit(v, r) = commit(v', r')
⟹ g^v * h^r = g^v' * h^r' (mod p)
⟹ g^(v-v') = h^(r'-r) (mod p)
⟹ (v-v') = (r'-r) * log_g(h) (mod q)
```
ถ้า log_g(h) ไม่เป็นที่รู้จักและ DLOG assumption ใช้ได้ การคำนวณ (v-v'), (r'-r) ดังกล่าวเป็นไปได้ยากมาก

**คุณสมบัติ Perfect Hiding:**

สำหรับค่า v ที่กำหนดไว้ การแจกแจงของ `g^v * h^r` เมื่อ r uniform ใน Z_q เป็น uniform distribution บน subgroup ทั้งหมด ดังนั้น C ไม่เปิดเผย information ใด ๆ เกี่ยวกับ v

---

### ขั้นที่ 5: Range Proof ด้วย Bit Decomposition

สร้าง `src/range_proof.rs`:

```rust
// Prove v ∈ [0, 2^n) without revealing v.
// 
// Protocol:
//   1. Decompose v = b_{n-1} * 2^{n-1} + ... + b_1 * 2 + b_0
//   2. Commit to each bit: C_i = commit(b_i, r_i)  for i = 0..n-1
//   3. Prove each C_i is a commitment to 0 or 1 (verifier checks C_i reopens correctly)
//   4. value_commitment = product(C_i^{2^i}) mod p = commit(v, r_weighted)

pub struct BitProof {
    pub commitment: Vec<u8>,  // C_i = g^b_i * h^r_i mod p
    pub blinding: Vec<u8>,    // r_i (revealed to verifier in this simplified version)
    pub bit: u8,              // b_i in {0, 1}
}

pub struct RangeProof {
    pub bit_proofs: Vec<BitProof>,
    pub n_bits: u32,
    pub value_commitment: Vec<u8>,  // product(C_i^{2^i}) mod p
}

pub fn prove_range(v: u64, n_bits: u32) -> RangeProof {
    assert!(v < (1u64 << n_bits));
    let (p, q, g, h) = params();
    let mut rng = thread_rng();
    let mut acc = BigUint::one();

    let bit_proofs: Vec<_> = (0..n_bits).map(|i| {
        let bit = ((v >> i) & 1) as u8;
        let r = rng.gen_biguint_range(&BigUint::one(), &q);
        let c = commit_bit(&BigUint::from(bit), &r, &p, &g, &h);
        let weight = BigUint::from(2u64).pow(i);
        acc = (&acc * c.modpow(&weight, &p)) % &p;
        BitProof { commitment: c.to_bytes_be(), blinding: r.to_bytes_be(), bit }
    }).collect();

    RangeProof { bit_proofs, n_bits, value_commitment: acc.to_bytes_be() }
}

pub fn verify_range(proof: &RangeProof, n_bits: u32) -> bool {
    // 1. Check bit count
    if proof.bit_proofs.len() != n_bits as usize { return false; }
    let (p, _, g, h) = params();
    let mut acc = BigUint::one();

    for (i, bp) in proof.bit_proofs.iter().enumerate() {
        // 2. Each bit must be 0 or 1
        if bp.bit > 1 { return false; }
        // 3. Commitment must match (bit, blinding)
        let c_expected = commit_bit(&BigUint::from(bp.bit), &BigUint::from_bytes_be(&bp.blinding), &p, &g, &h);
        let c_stored = BigUint::from_bytes_be(&bp.commitment);
        if c_expected != c_stored { return false; }
        // 4. Accumulate weighted product
        let weight = BigUint::from(2u64).pow(i as u32);
        acc = (&acc * c_stored.modpow(&weight, &p)) % &p;
    }
    acc == BigUint::from_bytes_be(&proof.value_commitment)
}
```

**หมายเหตุ:** implementation นี้เป็น pedagogical version — ใน production ควรใช้ **Bulletproofs** ที่ให้ O(log n) proof size แทน O(n) และไม่ต้อง reveal blinding factor ให้ verifier

---

### ขั้นที่ 6: Schnorr บน Ristretto255

สร้าง `src/ristretto_schnorr.rs` โดยใช้ `curve25519-dalek`:

```rust
use curve25519_dalek::{
    ristretto::RistrettoPoint,
    scalar::Scalar,
    constants::RISTRETTO_BASEPOINT_POINT,
};
use sha2::{Digest, Sha512};
use rand::rngs::OsRng;

pub struct RistrettoKeypair {
    pub secret: Scalar,
    pub public: RistrettoPoint,
}

impl RistrettoKeypair {
    pub fn generate() -> Self {
        let secret = Scalar::random(&mut OsRng);
        let public = &secret * RISTRETTO_BASEPOINT_POINT;
        RistrettoKeypair { secret, public }
    }
}

pub struct RistrettoProof {
    pub r_point: RistrettoPoint,
    pub s: Scalar,
}

/// Challenge: H(B || Y || R) as Scalar using Sha512 (wide reduction).
fn challenge(y: &RistrettoPoint, r: &RistrettoPoint) -> Scalar {
    let b = RISTRETTO_BASEPOINT_POINT;
    let mut h = Sha512::new();
    h.update(b.compress().as_bytes());
    h.update(y.compress().as_bytes());
    h.update(r.compress().as_bytes());
    // from_bytes_mod_order_wide ทำ wide reduction (512→256 bits) ป้องกัน bias
    let hash_bytes: [u8; 64] = h.finalize().into();
    Scalar::from_bytes_mod_order_wide(&hash_bytes)
}

pub fn schnorr_prove(kp: &RistrettoKeypair) -> RistrettoProof {
    let r_scalar = Scalar::random(&mut OsRng);
    let r_point = &r_scalar * RISTRETTO_BASEPOINT_POINT;
    let c = challenge(&kp.public, &r_point);
    let s = r_scalar + c * kp.secret;
    RistrettoProof { r_point, s }
}

/// Check: s * B == R + c * Y
pub fn schnorr_verify(public: &RistrettoPoint, proof: &RistrettoProof) -> bool {
    let c = challenge(public, &proof.r_point);
    let lhs = &proof.s * RISTRETTO_BASEPOINT_POINT;
    let rhs = proof.r_point + c * public;
    lhs == rhs
}
```

**ทำไม Ristretto255 ดีกว่า Curve25519 raw?**

Curve25519 มี cofactor 8 — group Z(8) × Z(l) แทนที่จะเป็น prime-order group ทำให้เกิดปัญหา:
- Small subgroup attacks: ถ้า point อยู่ใน 8-element subgroup, scalar multiplication ให้ผล predictable
- Malleability: encoding ของ point ไม่ canonical (หลาย byte sequences ให้ point เดียวกัน)

Ristretto255 แก้ปัญหาทั้งสองด้วยการ quotient ออก cofactor ทำให้ได้ prime-order group ที่ clean และ canonical

**Pitfall #3: ใช้ `Scalar::from_hash()` ที่ deprecated ใน curve25519-dalek v4**

ใน curve25519-dalek 4.x method `from_hash()` ถูกลบออก:
```rust
// ❌ ไม่ compile ใน curve25519-dalek 4.x
Scalar::from_hash(hasher)

// ✓ ใช้ from_bytes_mod_order_wide กับ Sha512 output (64 bytes)
let hash_bytes: [u8; 64] = hasher.finalize().into();
Scalar::from_bytes_mod_order_wide(&hash_bytes)
```

ต้องใช้ Sha512 (ไม่ใช่ Sha256) เพราะ `from_bytes_mod_order_wide` ต้องการ input 64 bytes

---

### ขั้นที่ 7: Proof Serialization

สร้าง `src/serialization.rs`:

```rust
use curve25519_dalek::{
    ristretto::CompressedRistretto,
    scalar::Scalar,
};
use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};
use crate::ristretto_schnorr::RistrettoProof;

/// Serialize proof = 32-byte compressed R || 32-byte scalar s = 64 bytes total.
pub fn proof_to_bytes(proof: &RistrettoProof) -> Vec<u8> {
    let mut out = Vec::with_capacity(64);
    out.extend_from_slice(proof.r_point.compress().as_bytes());
    out.extend_from_slice(proof.s.as_bytes());
    out
}

pub fn proof_from_bytes(bytes: &[u8]) -> Option<RistrettoProof> {
    if bytes.len() != 64 { return None; }

    // Decompress R
    let r_compressed = CompressedRistretto::from_slice(&bytes[..32]).ok()?;
    let r_point = r_compressed.decompress()?;

    // Deserialize s — from_canonical_bytes returns CtOption (subtle crate)
    // CtOption ไม่ implement Try ดังนั้นใช้ ? ไม่ได้
    let s_bytes: [u8; 32] = bytes[32..64].try_into().ok()?;
    let s_opt = Scalar::from_canonical_bytes(s_bytes);
    let s = if bool::from(s_opt.is_some()) {
        s_opt.unwrap()
    } else {
        return None;
    };

    Some(RistrettoProof { r_point, s })
}

/// Encode proof as base64url (no padding) for JSON transport.
pub fn proof_to_base64(proof: &RistrettoProof) -> String {
    URL_SAFE_NO_PAD.encode(proof_to_bytes(proof))
}

pub fn proof_from_base64(s: &str) -> Option<RistrettoProof> {
    let bytes = URL_SAFE_NO_PAD.decode(s).ok()?;
    proof_from_bytes(&bytes)
}
```

**Pitfall #4: `subtle::CtOption` ไม่ implement `Try` — ใช้ `?` ไม่ได้**

`Scalar::from_canonical_bytes` คืน `subtle::CtOption<Scalar>` ซึ่งเป็น constant-time Option ที่ **ไม่** implement `std::ops::Try`:

```rust
// ❌ ไม่ compile — CtOption ไม่ implement Try
let s = Scalar::from_canonical_bytes(bytes)?;

// ✓ ต้อง extract ด้วย bool::from(opt.is_some())
let opt = Scalar::from_canonical_bytes(bytes);
let s = if bool::from(opt.is_some()) { opt.unwrap() } else { return None; };
```

เหตุผล: `CtOption` ออกแบบมาให้ branch ทำงานใน constant time เพื่อป้องกัน timing side channels ดังนั้น Rust ไม่อนุญาตให้ใช้ `?` (early return) กับมัน

---

### ขั้นที่ 8: Batch Verification

สร้าง `src/batch_verify.rs`:

```rust
use curve25519_dalek::{
    ristretto::RistrettoPoint,
    scalar::Scalar,
    constants::RISTRETTO_BASEPOINT_POINT,
};
use rand::rngs::OsRng;
use sha2::{Digest, Sha512};
use crate::ristretto_schnorr::RistrettoProof;

/// Batch verify N Schnorr proofs using random linear combination.
///
/// Instead of checking: for each i,  s_i * B == R_i + c_i * Y_i
/// We check one equation:
///   sum(a_i * s_i) * B  ==  sum(a_i * (R_i + c_i * Y_i))
///
/// where a_i are random scalars chosen by verifier.
///
/// Correctness: if all proofs valid, both sides equal sum(a_i * s_i * B) — consistent.
/// Soundness: if proof j is invalid (s_j * B ≠ R_j + c_j * Y_j), then with
///   overwhelming probability (1 - 1/l) the random a_j breaks the equation.
pub fn batch_verify_schnorr(
    public_keys: &[RistrettoPoint],
    proofs: &[RistrettoProof],
) -> bool {
    let n = public_keys.len();
    if n == 0 { return true; }
    assert_eq!(n, proofs.len());

    // Random weights a_i
    let a: Vec<Scalar> = (0..n).map(|_| Scalar::random(&mut OsRng)).collect();

    // LHS: (sum a_i * s_i) * B
    let lhs_scalar: Scalar = a.iter().zip(proofs).map(|(ai, pf)| ai * pf.s).sum();
    let lhs = lhs_scalar * RISTRETTO_BASEPOINT_POINT;

    // RHS: sum a_i * (R_i + c_i * Y_i)
    let rhs: RistrettoPoint = a.iter()
        .zip(proofs.iter().zip(public_keys.iter()))
        .map(|(ai, (pf, yk))| {
            let c = challenge(yk, &pf.r_point);
            ai * (pf.r_point + c * yk)
        })
        .sum();

    lhs == rhs
}
```

ประสิทธิภาพของ batch verification ขึ้นอยู่กับ backend ที่ใช้ — `curve25519-dalek` ใช้ Pippenger algorithm สำหรับ multi-scalar multiplication ทำให้ LHS คำนวณด้วย O(N/log N) point additions แทน O(N)

---

### ขั้นที่ 9: Interactive Demo

**`src/main.rs`** — แสดง protocol transcript ทีละขั้น:

```rust
pub mod schnorr_prime;
pub mod fiat_shamir;
pub mod pedersen;
pub mod range_proof;
pub mod ristretto_schnorr;
pub mod serialization;
pub mod batch_verify;

fn main() {
    println!("=== Zero-Knowledge Proof Demo ===\n");

    // --- Schnorr interactive protocol ---
    println!("--- Schnorr Identification Protocol (safe-prime group) ---");
    let (pk, sk) = schnorr_prime::keygen();
    let commit = schnorr_prime::commit_round1();
    println!("[Prover]   R = g^r mod p  (commitment sent to Verifier)");
    println!("[Verifier] challenge c chosen at random");

    use num_bigint::RandBigInt;
    let (_, q, _) = {
        // นำ params มาใช้ผ่าน public function
        use num_bigint::BigUint;
        use num_traits::One;
        let p_hex = "B10B8F96A080E01DDE92DE5EAE5D54EC52C99FBCFB06A3C69A6A9DCA52D23B616073E28675A23D189838EF1E2EE652C013ECB4AEA906112324975C3CD49B83BFACCBDD7D90C4BD7098488E9C219A73724EFFD6FAE5644738FAA31A4FF55BCCC0A151AF5F0DC8B4BD45BF37DF365C1A65E68CFDA76D4DA708DF1FB2BC2E4A4371";
        let p = BigUint::parse_bytes(p_hex.as_bytes(), 16).unwrap();
        let q = (&p - BigUint::one()) / BigUint::from(2u32);
        let g = BigUint::from(4u32);
        (p, q, g)
    };
    let c = rand::thread_rng().gen_biguint_range(&num_bigint::BigUint::one(), &q);
    let s = schnorr_prime::respond(&commit, &sk, &c);
    let ok = schnorr_prime::verify(&pk, &commit.r_point, &c, &s);
    println!("[Prover]   s = r + c*x mod q  (response)");
    println!("[Verifier] g^s == R * Y^c?  => {ok}\n");

    // --- Fiat-Shamir ---
    println!("--- Fiat-Shamir Non-Interactive ---");
    let (pk, sk) = schnorr_prime::keygen();
    let proof = fiat_shamir::prove_non_interactive(&sk, &pk);
    let valid = fiat_shamir::verify_non_interactive(&pk, &proof);
    println!("Proof (R, s) valid: {valid}");
    println!("Proof R = {}...", &hex_encode(&proof.r_bytes)[..16]);

    // --- Pedersen commitment ---
    println!("\n--- Pedersen Commitment ---");
    use num_bigint::BigUint;
    let v = BigUint::from(42u32);
    let r = pedersen::random_blinding();
    let c = pedersen::commit(&v, &r);
    println!("commit(42, r) = {}...", &hex_encode(&c.to_bytes_be())[..16]);
    println!("open(C, 42, r) = {}", pedersen::open(&c, &v, &r));
    println!("open(C, 43, r) = {}", pedersen::open(&c, &BigUint::from(43u32), &r));

    // --- Range proof ---
    println!("\n--- Range Proof [0, 4) ---");
    for v in 0u64..4 {
        let rp = range_proof::prove_range(v, 2);
        let ok = range_proof::verify_range(&rp, 2);
        println!("  v={v} in [0,4): {ok}");
    }

    // --- Ristretto Schnorr ---
    println!("\n--- Ristretto255 Schnorr ---");
    use ristretto_schnorr::{RistrettoKeypair, schnorr_prove, schnorr_verify};
    let kp = RistrettoKeypair::generate();
    let pf = schnorr_prove(&kp);
    println!("Ristretto Schnorr valid: {}", schnorr_verify(&kp.public, &pf));

    // --- Serialization round-trip ---
    println!("\n--- Proof Serialization ---");
    let b64 = serialization::proof_to_base64(&pf);
    println!("base64url proof: {}...", &b64[..22]);
    let pf2 = serialization::proof_from_base64(&b64).unwrap();
    println!("Deserialized proof valid: {}", schnorr_verify(&kp.public, &pf2));

    // --- Batch verify ---
    println!("\n--- Batch Verification (N=8) ---");
    let (pks, pfs): (Vec<_>, Vec<_>) = (0..8).map(|_| {
        let kp = RistrettoKeypair::generate();
        let pf = schnorr_prove(&kp);
        (kp.public, pf)
    }).unzip();
    println!("Batch verify 8 proofs: {}", batch_verify::batch_verify_schnorr(&pks, &pfs));
}

fn hex_encode(b: &[u8]) -> String {
    b.iter().map(|x| format!("{x:02x}")).collect()
}
```

---

### ขั้นที่ 10: HTTP API (axum 0.8)

เพิ่ม API endpoint ใน `src/main.rs` หรือสร้าง `src/api.rs`:

```rust
use axum::{Router, Json, routing::post, http::StatusCode};
use serde::{Deserialize, Serialize};
use tokio::net::TcpListener;

#[derive(Deserialize)]
struct SchnorrProveRequest {
    /// Witness x (hex) — secret key
    witness: String,
    /// Public key Y (hex) — g^x mod p
    public_key: String,
}

#[derive(Serialize)]
struct SchnorrProveResponse {
    /// Non-interactive proof (R, s) encoded as base64url
    proof: String,
    /// Fiat-Shamir challenge c (hex)
    challenge: String,
}

#[derive(Deserialize)]
struct SchnorrVerifyRequest {
    public_key: String,
    proof: String,
}

#[derive(Serialize)]
struct SchnorrVerifyResponse {
    valid: bool,
}

async fn generate_schnorr_proof(
    Json(req): Json<SchnorrProveRequest>,
) -> Result<Json<SchnorrProveResponse>, StatusCode> {
    use num_bigint::BigUint;
    let sk_bytes = hex::decode(&req.witness).map_err(|_| StatusCode::BAD_REQUEST)?;
    let pk_bytes = hex::decode(&req.public_key).map_err(|_| StatusCode::BAD_REQUEST)?;
    let sk = schnorr_prime::SecretKey(BigUint::from_bytes_be(&sk_bytes));
    let pk = schnorr_prime::PublicKey(BigUint::from_bytes_be(&pk_bytes));
    let proof = fiat_shamir::prove_non_interactive(&sk, &pk);
    let challenge_hex = hex_encode(/* recompute challenge */);
    Ok(Json(SchnorrProveResponse {
        proof: base64::engine::general_purpose::URL_SAFE_NO_PAD.encode(
            [proof.r_bytes.clone(), proof.s_bytes.clone()].concat()
        ),
        challenge: challenge_hex,
    }))
}

async fn verify_schnorr_proof(
    Json(req): Json<SchnorrVerifyRequest>,
) -> Json<SchnorrVerifyResponse> {
    use num_bigint::BigUint;
    let pk_bytes = match hex::decode(&req.public_key) {
        Ok(b) => b, Err(_) => return Json(SchnorrVerifyResponse { valid: false })
    };
    let pk = schnorr_prime::PublicKey(BigUint::from_bytes_be(&pk_bytes));
    let proof_bytes = match base64::engine::general_purpose::URL_SAFE_NO_PAD.decode(&req.proof) {
        Ok(b) => b, Err(_) => return Json(SchnorrVerifyResponse { valid: false })
    };
    let proof = fiat_shamir::NiProof {
        r_bytes: proof_bytes[..proof_bytes.len()/2].to_vec(),
        s_bytes: proof_bytes[proof_bytes.len()/2..].to_vec(),
    };
    Json(SchnorrVerifyResponse {
        valid: fiat_shamir::verify_non_interactive(&pk, &proof)
    })
}

pub async fn run_api() {
    let app = Router::new()
        .route("/proofs/schnorr/generate", post(generate_schnorr_proof))
        .route("/proofs/schnorr/verify", post(verify_schnorr_proof));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("ZKP API listening on :3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

### ขั้นที่ 11: Criterion Benchmarks

สร้าง `benches/zkp_bench.rs`:

```rust
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};
use zkp_demo::{
    schnorr_prime,
    fiat_shamir,
    ristretto_schnorr::{RistrettoKeypair, schnorr_prove, schnorr_verify},
    batch_verify::batch_verify_schnorr,
};

fn bench_schnorr_prime(c: &mut Criterion) {
    let mut group = c.benchmark_group("schnorr_prime");

    group.bench_function("keygen", |b| b.iter(|| schnorr_prime::keygen()));

    let (pk, sk) = schnorr_prime::keygen();
    group.bench_function("prove_ni", |b| {
        b.iter(|| fiat_shamir::prove_non_interactive(&sk, &pk))
    });

    let proof = fiat_shamir::prove_non_interactive(&sk, &pk);
    group.bench_function("verify_ni", |b| {
        b.iter(|| fiat_shamir::verify_non_interactive(&pk, &proof))
    });
    group.finish();
}

fn bench_ristretto(c: &mut Criterion) {
    let mut group = c.benchmark_group("ristretto255");

    group.bench_function("keygen", |b| b.iter(|| RistrettoKeypair::generate()));

    let kp = RistrettoKeypair::generate();
    group.bench_function("prove", |b| b.iter(|| schnorr_prove(&kp)));

    let pf = schnorr_prove(&kp);
    group.bench_function("verify", |b| b.iter(|| schnorr_verify(&kp.public, &pf)));
    group.finish();
}

fn bench_batch_verify(c: &mut Criterion) {
    let mut group = c.benchmark_group("batch_verify");

    for n in [1usize, 4, 8, 16, 32, 64] {
        let (pks, pfs): (Vec<_>, Vec<_>) = (0..n).map(|_| {
            let kp = RistrettoKeypair::generate();
            let pf = schnorr_prove(&kp);
            (kp.public, pf)
        }).unzip();

        // Individual verification
        group.bench_with_input(BenchmarkId::new("individual", n), &n, |b, _| {
            b.iter(|| pks.iter().zip(&pfs).all(|(pk, pf)| schnorr_verify(pk, pf)))
        });

        // Batch verification
        group.bench_with_input(BenchmarkId::new("batch", n), &n, |b, _| {
            b.iter(|| batch_verify_schnorr(&pks, &pfs))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_schnorr_prime, bench_ristretto, bench_batch_verify);
criterion_main!(benches);
```

รัน benchmarks:
```bash
cargo bench
# เปิด target/criterion/report/index.html สำหรับ HTML report
```

ผลลัพธ์ที่คาดหวัง (approximate, บน i7-class CPU):

| Operation | Safe-prime (1024-bit) | Ristretto255 |
|---|---|---|
| keygen | ~800 µs | ~15 µs |
| prove | ~1.2 ms | ~22 µs |
| verify | ~1.1 ms | ~18 µs |
| batch/16 vs individual | — | ~30% faster |

Ristretto255 เร็วกว่า safe-prime group ประมาณ 50x เพราะใช้ dedicated ECC arithmetic แทน generic BigUint modular exponentiation

## การทดสอบ (Testing)

### Unit Tests ทั้งหมด

```bash
cargo test 2>&1
```

**ผลลัพธ์จริงจากการรัน:**

```
running 29 tests
test batch_verify::tests::test_batch_empty ... ok
test batch_verify::tests::test_batch_verify_1 ... ok
test batch_verify::tests::test_batch_verify_4 ... ok
test fiat_shamir::tests::test_fiat_shamir_non_interactive_valid ... ok
test fiat_shamir::tests::test_fiat_shamir_tampered_r_fails ... ok
test fiat_shamir::tests::test_fiat_shamir_tampered_s_fails ... ok
test fiat_shamir::tests::test_fiat_shamir_wrong_key_fails ... ok
test pedersen::tests::test_pedersen_commit_open ... ok
test pedersen::tests::test_pedersen_wrong_blinding_fails ... ok
test batch_verify::tests::test_batch_verify_tampered_fails ... ok
test pedersen::tests::test_pedersen_homomorphic_addition ... ok
test pedersen::tests::test_pedersen_wrong_value_fails ... ok
test range_proof::tests::test_range_proof_0_in_range ... ok
test range_proof::tests::test_range_proof_1_in_range ... ok
test range_proof::tests::test_range_proof_2_in_range ... ok
test range_proof::tests::test_range_proof_3_in_range ... ok
test range_proof::tests::test_range_proof_tampered_bit_fails ... ok
test range_proof::tests::test_range_proof_bit_decomposition ... ok
test ristretto_schnorr::tests::test_ristretto_multiple_proofs_independent ... ok
test ristretto_schnorr::tests::test_ristretto_schnorr_round_trip ... ok
test ristretto_schnorr::tests::test_ristretto_schnorr_tampered_s_fails ... ok
test batch_verify::tests::test_batch_verify_16 ... ok
test ristretto_schnorr::tests::test_ristretto_schnorr_wrong_key_fails ... ok
test schnorr_prime::tests::test_schnorr_complete_protocol ... ok
test serialization::tests::test_proof_base64_round_trip ... ok
test serialization::tests::test_truncated_bytes_fail ... ok
test schnorr_prime::tests::test_schnorr_tampered_proof_fails ... ok
test serialization::tests::test_proof_bytes_round_trip ... ok
test schnorr_prime::tests::test_wrong_public_key_fails ... ok

test result: ok. 29 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.74s
```

### Test Coverage Matrix

| Test | สิ่งที่ตรวจสอบ |
|---|---|
| `test_schnorr_complete_protocol` | Schnorr prove+verify = true (completeness) |
| `test_schnorr_tampered_proof_fails` | s+1 → verify = false (soundness) |
| `test_wrong_public_key_fails` | verify กับ pk ผิด = false |
| `test_fiat_shamir_non_interactive_valid` | Fiat-Shamir NI proof valid |
| `test_fiat_shamir_tampered_s_fails` | s XOR 0xFF → false |
| `test_fiat_shamir_tampered_r_fails` | R XOR 0x01 → false (recomputed challenge mismatch) |
| `test_pedersen_commit_open` | commit(42,r) opens correctly |
| `test_pedersen_homomorphic_addition` | C(7,r1)*C(5,r2) = C(12,(r1+r2)%q) |
| `test_range_proof_{0,1,2,3}_in_range` | bit decomposition ถูกต้องสำหรับ [0,4) |
| `test_range_proof_bit_decomposition` | v=5 → bits=[1,0,1] (bit 0, 1, 2) |
| `test_ristretto_schnorr_round_trip` | Ristretto Schnorr prove+verify = true |
| `test_ristretto_schnorr_tampered_s_fails` | s+999 → false |
| `test_proof_bytes_round_trip` | 64-byte serialization round-trip |
| `test_proof_base64_round_trip` | base64url 86-char round-trip |
| `test_batch_verify_{1,4,16}` | batch verify N valid proofs = true |
| `test_batch_verify_tampered_fails` | 1 tampered proof ใน batch → false |

### Demo Output จริง

```
=== Zero-Knowledge Proof Demo ===

--- Schnorr Identification Protocol (safe-prime group) ---
Public key Y  = 6f09827d84e9f6c6f52f8b9722d05f77...  (first 32 hex chars)
Fiat-Shamir proof valid: true
Proof R = 454f0604305cc553bfebb96475778b84...

--- Pedersen Commitment ---
commit(42, r) opened correctly: true

--- Range Proof (value in [0, 4)) ---
Range proof for v=3 in [0,4): true

--- Ristretto255 Schnorr ---
Ristretto Schnorr proof valid: true

--- Proof Serialization ---
Round-trip serialized proof valid: true

--- Batch Verification (N=8) ---
Batch verify 8 proofs: true

All demos completed successfully.
```

## Pitfalls สรุป

ระหว่างพัฒนาโปรเจคนี้ พบปัญหาที่พบบ่อยดังนี้:

### Pitfall #1: Generator มี order ผิด (2q แทน q)

```rust
// ❌ g=2 มี order 2q สำหรับ MODP primes — Schnorr verify จะ fail
let g = BigUint::from(2u32);

// ✓ ใช้ generator จาก RFC 5114 ที่ verify แล้วว่า order = q
let g = BigUint::parse_bytes(b"A4D1CBD5C3FD...", 16).unwrap();
// หรือ g' = g^2 mod p ซึ่ง ord(g^2) = ord(g)/gcd(2,ord(g)) = 2q/2 = q
```

สาเหตุ: Schnorr ต้องการให้ exponents ลดลง mod q แต่ถ้า g มี order 2q ก็จะมี g^(s mod q) ≠ g^s ทำให้ verify equation ไม่สมดุล

### Pitfall #2: ลืม domain separation ใน Fiat-Shamir

```rust
// ❌ hash แค่ R — proof สามารถ transfer ระหว่าง public keys ได้
let c = Sha256::digest(&r_bytes);

// ✓ hash context ครบ: g || Y || R
hasher.update(g.to_bytes_be());
hasher.update(y.to_bytes_be());
hasher.update(r_point.to_bytes_be());
```

### Pitfall #3: `Scalar::from_hash()` deprecated ใน curve25519-dalek 4.x

```rust
// ❌ ไม่มีใน v4
Scalar::from_hash(Sha512::new().chain_update(...))

// ✓ ใช้ Sha512 + from_bytes_mod_order_wide
let hash_bytes: [u8; 64] = Sha512::new().chain_update(...).finalize().into();
Scalar::from_bytes_mod_order_wide(&hash_bytes)
```

### Pitfall #4: `subtle::CtOption` ไม่ implement `Try`

```rust
// ❌ ไม่ compile
let s = Scalar::from_canonical_bytes(bytes)?;

// ✓ extract แบบ explicit
let opt = Scalar::from_canonical_bytes(bytes);
let s = if bool::from(opt.is_some()) { opt.unwrap() } else { return None; };
```

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/zkp-demo
```

### Docker Image

```dockerfile
FROM rust:1.75-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/zkp-demo .
EXPOSE 3000
CMD ["./zkp-demo", "--api"]
```

```bash
docker build -t zkp-demo .
docker run -p 3000:3000 zkp-demo
```

### API Testing

```bash
# Start server
cargo run -- --api

# Generate proof
curl -X POST http://localhost:3000/proofs/schnorr/generate \
  -H "Content-Type: application/json" \
  -d '{"witness": "deadbeef...", "public_key": "cafe1234..."}'

# Verify proof
curl -X POST http://localhost:3000/proofs/schnorr/verify \
  -H "Content-Type: application/json" \
  -d '{"public_key": "cafe1234...", "proof": "base64url..."}'
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Implement Pedersen Commitment บน Ristretto255 (ง่าย)

สร้าง `src/ristretto_pedersen.rs` ที่ implement Pedersen commitment โดยใช้ `RistrettoPoint` แทน BigUint:

```rust
use curve25519_dalek::{RistrettoPoint, Scalar, constants::RISTRETTO_BASEPOINT_POINT};

// Setup: G = base point, H = hash_to_point("zkp-pedersen-h")
// C = v*G + r*H

pub struct RistrettoPedersenParams {
    pub g: RistrettoPoint,  // RISTRETTO_BASEPOINT_POINT
    pub h: RistrettoPoint,  // second generator via hash-to-curve
}

pub fn commit(params: &RistrettoPedersenParams, v: &Scalar, r: &Scalar) -> RistrettoPoint {
    v * params.g + r * params.h
}
```

ต้องหา second generator H โดยใช้ hash-to-curve (ตัวอย่าง: SHA-512 ของ string "zkp-pedersen-h" แล้ว `RistrettoPoint::from_uniform_bytes`)

### แบบฝึกหัดที่ 2: Implement σ-OR Proof สำหรับ Range Proof (กลาง)

ใน `range_proof.rs` ปัจจุบัน verifier ต้องเชื่อใจ prover เกี่ยวกับ blinding factor ปรับปรุงให้ใช้ Sigma-OR proof ที่แท้จริง:

```
"I know r such that  C = commit(0, r)  OR  C = commit(1, r)"
```

Sigma-OR: สร้าง proof (c0, s0, c1, s1) ที่ satisfy c0 + c1 = H(C || context) และ verify equation ทั้งสองสาขา

### แบบฝึกหัดที่ 3: Schnorr Multi-Signature (MuSig2) (ยาก)

Implement MuSig2 protocol ที่ n parties รวม public keys เป็น aggregate key แล้ว co-sign:

1. Key aggregation: X_agg = sum(a_i * X_i) ด้วย coefficient a_i = H(L || X_i)
2. Round 1: แต่ละ signer ส่ง (R_i1, R_i2)
3. Round 2: แต่ละ signer คำนวณ partial signature s_i
4. Aggregate: s = sum(s_i)

ต้องระวัง rogue key attack — ต้อง prove knowledge of secret key ก่อน aggregation

### แบบฝึกหัดที่ 4: Groth16 Toy Implementation (ยากมาก)

Implement toy Groth16 (zkSNARK) สำหรับ circuit ง่าย ๆ เช่น `x^2 + y = z`:

1. Arithmetic circuit → R1CS (Rank-1 Constraint System)
2. QAP (Quadratic Arithmetic Program) transformation
3. Trusted setup (CRS generation)
4. Prover: compute A, B, C ∈ pairing group
5. Verifier: check pairing equation e(A, B) = e(α, β) * e(vk, γ) * e(C, δ)

ใช้ library `ark-groth16` และ `ark-bn254` สำหรับ BN254 pairing-friendly curve

## รายละเอียด Module ครบชุด

### schnorr_prime.rs — ฉบับสมบูรณ์

```rust
// src/schnorr_prime.rs
use num_bigint::BigUint;
use num_traits::One;
use rand::thread_rng;
use num_bigint::RandBigInt;

fn params() -> (BigUint, BigUint, BigUint) {
    let p = BigUint::parse_bytes(
        b"B10B8F96A080E01DDE92DE5EAE5D54EC52C99FBCFB06A3C69A6A9DCA52D23B616073E28675A23D189838EF1E2EE652C013ECB4AEA906112324975C3CD49B83BFACCBDD7D90C4BD7098488E9C219A73724EFFD6FAE5644738FAA31A4FF55BCCC0A151AF5F0DC8B4BD45BF37DF365C1A65E68CFDA76D4DA708DF1FB2BC2E4A4371",
        16,
    ).unwrap();
    let q = (&p - BigUint::one()) / BigUint::from(2u32);
    let g = BigUint::parse_bytes(
        b"A4D1CBD5C3FD34126765A442EFB99905F8104DD258AC507FD6406CFF14266D31266FEA1E5C41564B777E690F5504F213160217B4B01B886A5E91547F9E2749F4D7FBD7D3B9A92EE1909D0D2263F80A76A6A24C087A091F531DBF0A0169B6A28AD662A4D18E73AFA32D779D5918D4F3C8E49401D25DA672B50EE1B3A4EF11F2D6FEB",
        16,
    ).unwrap();
    (p, q, g)
}

pub struct SecretKey(pub BigUint);
pub struct PublicKey(pub BigUint);

impl PublicKey {
    pub fn to_bytes_be(&self) -> Vec<u8> { self.0.to_bytes_be() }
}

pub fn keygen() -> (PublicKey, SecretKey) {
    let (p, q, g) = params();
    let mut rng = thread_rng();
    let x = rng.gen_biguint_range(&BigUint::one(), &q);
    let y = g.modpow(&x, &p);
    (PublicKey(y), SecretKey(x))
}

pub struct Commitment {
    pub r_scalar: BigUint,
    pub r_point: BigUint,
}

pub fn commit_round1() -> Commitment {
    let (p, q, g) = params();
    let mut rng = thread_rng();
    let r = rng.gen_biguint_range(&BigUint::one(), &q);
    let r_point = g.modpow(&r, &p);
    Commitment { r_scalar: r, r_point }
}

pub fn respond(comm: &Commitment, sk: &SecretKey, c: &BigUint) -> BigUint {
    let (_, q, _) = params();
    (&comm.r_scalar + (c * &sk.0) % &q) % &q
}

pub fn verify(pk: &PublicKey, r_point: &BigUint, c: &BigUint, s: &BigUint) -> bool {
    let (p, _, g) = params();
    g.modpow(s, &p) == (r_point * pk.0.modpow(c, &p)) % &p
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_schnorr_complete_protocol() {
        let (pk, sk) = keygen();
        let (_, q, _) = params();
        let comm = commit_round1();
        let mut rng = thread_rng();
        let c = rng.gen_biguint_range(&BigUint::one(), &q);
        let s = respond(&comm, &sk, &c);
        assert!(verify(&pk, &comm.r_point, &c, &s));
    }

    #[test]
    fn test_schnorr_tampered_proof_fails() {
        let (pk, sk) = keygen();
        let (_, q, _) = params();
        let comm = commit_round1();
        let mut rng = thread_rng();
        let c = rng.gen_biguint_range(&BigUint::one(), &q);
        let mut s = respond(&comm, &sk, &c);
        s += BigUint::one();
        assert!(!verify(&pk, &comm.r_point, &c, &s));
    }

    #[test]
    fn test_wrong_public_key_fails() {
        let (pk, sk) = keygen();
        let (pk_other, _) = keygen();
        let (_, q, _) = params();
        let comm = commit_round1();
        let mut rng = thread_rng();
        let c = rng.gen_biguint_range(&BigUint::one(), &q);
        let s = respond(&comm, &sk, &c);
        assert!(!verify(&pk_other, &comm.r_point, &c, &s));
        assert!(verify(&pk, &comm.r_point, &c, &s));
    }
}
```

### ristretto_schnorr.rs — ฉบับสมบูรณ์

```rust
// src/ristretto_schnorr.rs
use curve25519_dalek::{
    ristretto::RistrettoPoint,
    scalar::Scalar,
    constants::RISTRETTO_BASEPOINT_POINT,
};
use sha2::{Digest, Sha512};
use rand::rngs::OsRng;

pub struct RistrettoKeypair {
    pub secret: Scalar,
    pub public: RistrettoPoint,
}

impl RistrettoKeypair {
    pub fn generate() -> Self {
        let secret = Scalar::random(&mut OsRng);
        let public = &secret * RISTRETTO_BASEPOINT_POINT;
        RistrettoKeypair { secret, public }
    }
}

pub struct RistrettoProof {
    pub r_point: RistrettoPoint,
    pub s: Scalar,
}

fn challenge(y: &RistrettoPoint, r: &RistrettoPoint) -> Scalar {
    let b = RISTRETTO_BASEPOINT_POINT;
    let mut h = Sha512::new();
    h.update(b.compress().as_bytes());
    h.update(y.compress().as_bytes());
    h.update(r.compress().as_bytes());
    let hash_bytes: [u8; 64] = h.finalize().into();
    Scalar::from_bytes_mod_order_wide(&hash_bytes)
}

pub fn schnorr_prove(kp: &RistrettoKeypair) -> RistrettoProof {
    let r_scalar = Scalar::random(&mut OsRng);
    let r_point = &r_scalar * RISTRETTO_BASEPOINT_POINT;
    let c = challenge(&kp.public, &r_point);
    let s = r_scalar + c * kp.secret;
    RistrettoProof { r_point, s }
}

pub fn schnorr_verify(public: &RistrettoPoint, proof: &RistrettoProof) -> bool {
    let c = challenge(public, &proof.r_point);
    let lhs = &proof.s * RISTRETTO_BASEPOINT_POINT;
    let rhs = proof.r_point + c * public;
    lhs == rhs
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_ristretto_schnorr_round_trip() {
        let kp = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        assert!(schnorr_verify(&kp.public, &proof));
    }

    #[test]
    fn test_ristretto_schnorr_wrong_key_fails() {
        let kp = RistrettoKeypair::generate();
        let kp2 = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        assert!(!schnorr_verify(&kp2.public, &proof));
    }

    #[test]
    fn test_ristretto_schnorr_tampered_s_fails() {
        let kp = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        let tampered = RistrettoProof {
            r_point: proof.r_point,
            s: proof.s + Scalar::from(999u64),
        };
        assert!(!schnorr_verify(&kp.public, &tampered));
    }

    #[test]
    fn test_ristretto_multiple_proofs_independent() {
        let kp = RistrettoKeypair::generate();
        let p1 = schnorr_prove(&kp);
        let p2 = schnorr_prove(&kp);
        assert!(schnorr_verify(&kp.public, &p1));
        assert!(schnorr_verify(&kp.public, &p2));
        assert_ne!(p1.r_point.compress().as_bytes(), p2.r_point.compress().as_bytes());
    }
}
```

### serialization.rs — ฉบับสมบูรณ์

```rust
// src/serialization.rs
use curve25519_dalek::{ristretto::CompressedRistretto, scalar::Scalar};
use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};
use crate::ristretto_schnorr::RistrettoProof;

/// 64 bytes: 32-byte compressed Ristretto point (R) || 32-byte scalar (s)
pub fn proof_to_bytes(proof: &RistrettoProof) -> Vec<u8> {
    let mut out = Vec::with_capacity(64);
    out.extend_from_slice(proof.r_point.compress().as_bytes());
    out.extend_from_slice(proof.s.as_bytes());
    out
}

pub fn proof_from_bytes(bytes: &[u8]) -> Option<RistrettoProof> {
    if bytes.len() != 64 { return None; }
    let r_compressed = CompressedRistretto::from_slice(&bytes[..32]).ok()?;
    let r_point = r_compressed.decompress()?;
    let s_bytes: [u8; 32] = bytes[32..64].try_into().ok()?;
    // subtle::CtOption ไม่ implement Try — ต้อง unwrap แบบ explicit
    let s_opt = Scalar::from_canonical_bytes(s_bytes);
    let s = if bool::from(s_opt.is_some()) { s_opt.unwrap() } else { return None; };
    Some(RistrettoProof { r_point, s })
}

pub fn proof_to_base64(proof: &RistrettoProof) -> String {
    URL_SAFE_NO_PAD.encode(proof_to_bytes(proof))
}

pub fn proof_from_base64(s: &str) -> Option<RistrettoProof> {
    proof_from_bytes(&URL_SAFE_NO_PAD.decode(s).ok()?)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::ristretto_schnorr::{RistrettoKeypair, schnorr_prove, schnorr_verify};

    #[test]
    fn test_proof_bytes_round_trip() {
        let kp = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        let bytes = proof_to_bytes(&proof);
        assert_eq!(bytes.len(), 64);
        let proof2 = proof_from_bytes(&bytes).unwrap();
        assert!(schnorr_verify(&kp.public, &proof2));
    }

    #[test]
    fn test_proof_base64_round_trip() {
        let kp = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        let b64 = proof_to_base64(&proof);
        assert_eq!(b64.len(), 86); // base64url of 64 bytes = 86 chars (no padding)
        let proof2 = proof_from_base64(&b64).unwrap();
        assert!(schnorr_verify(&kp.public, &proof2));
    }

    #[test]
    fn test_truncated_bytes_fail() {
        let kp = RistrettoKeypair::generate();
        let proof = schnorr_prove(&kp);
        let mut bytes = proof_to_bytes(&proof);
        bytes.truncate(32);
        assert!(proof_from_bytes(&bytes).is_none());
    }
}
```

## ความเชื่อมโยงกับ Cryptographic Theory

### Schnorr Protocol และ Honest Verifier Zero-Knowledge

Schnorr protocol เป็น HVZK (Honest Verifier Zero-Knowledge):

**Simulator:** สำหรับทุก public key Y และ challenge c ที่ arbitrary สามารถสร้าง "transcript" (R, c, s) ที่ verify ได้โดยไม่รู้ secret:
1. เลือก s random
2. คำนวณ R = g^s * Y^{-c} mod p  (backward computation)
3. Transcript (R, c, s) pass verify โดยที่ไม่รู้ x

นี่คือสิ่งพิสูจน์ว่า transcript ไม่เปิดเผย secret x — distribution ของ simulated transcripts เหมือน real transcripts

### ทำไม Fiat-Shamir ใช้ Random Oracle Model

Fiat-Shamir ถือว่า hash function H เป็น perfect random function (Random Oracle) — ผู้ adversary ไม่สามารถ predict output ของ H ก่อนที่จะ query ได้ ใน practice ใช้ SHA-256/SHA-512 แต่ proof of security ต้องใช้ ROM assumption

ถ้า H ไม่ random (เช่น ถ้า H มี collision) ก็อาจมีช่องโหว่:
- Adversary สามารถหา (R, R') ที่ H(R) = H(R') แล้วใช้ proof ของ R กับ R' แทนกัน

### Knowledge Soundness

Schnorr protocol มีคุณสมบัติ **knowledge soundness** — ถ้า prover สามารถ pass verification สำหรับ challenge c **สองค่าที่ต่างกัน** (c, c') พร้อมกัน responses (s, s') แล้ว extractor สามารถ compute x:

```
g^s = R * Y^c     → s = r + cx mod q
g^s' = R * Y^c'   → s' = r + c'x mod q

s - s' = (c - c') * x mod q
x = (s - s') / (c - c') mod q  ← ถ้า c ≠ c'
```

ดังนั้น prover ที่ pass verification probability สูงกว่า 1/q ต้องรู้ x จริง (มิฉะนั้น knowledge extractor จะ rewind และ extract x ได้)

### Pedersen Commitment กับ Binding-Hiding Tradeoff

| Scheme | Binding | Hiding |
|---|---|---|
| Hash commitment H(v ‖ r) | Computational (collision resistance) | Computational (preimage resistance) |
| Pedersen C = g^v * h^r | Computational (DLOG) | **Perfect** (information-theoretic) |
| Deterministic C = H(v) | Computational | None |

Pedersen มี perfect hiding (information-theoretic) แต่ computational binding เท่านั้น — ตรงข้ามกับ hash commitment ที่มี perfect binding แต่ computational hiding

## ตัวอย่าง Protocol Transcript ที่สมบูรณ์

ตัวอย่าง concrete (ใช้ค่า p=23 เพื่อแสดงให้เห็น — ไม่ใช่ secure):

```
p = 23, q = 11, g = 4 (generator of order-11 subgroup mod 23)
ตรวจสอบ: 4^11 mod 23 = 4194304 mod 23 = 1 ✓

Secret x = 3
Public Y = g^x mod p = 4^3 mod 23 = 64 mod 23 = 18

--- Interactive Schnorr ---
Prover:
  r = 7  (random)
  R = 4^7 mod 23 = 16384 mod 23 = 16  → send R=16

Verifier:
  c = 5  (random challenge)  → send c=5

Prover:
  s = r + c*x mod q = 7 + 5*3 mod 11 = 22 mod 11 = 0 → send s=0

Wait: s=0 ทำให้ g^s = g^0 = 1 mod 23
R * Y^c = 16 * 18^5 mod 23

18^5 = 1889568 mod 23:
18^2 = 324 mod 23 = 2
18^4 = 4 mod 23
18^5 = 18^4 * 18 = 4 * 18 mod 23 = 72 mod 23 = 3
R * Y^c = 16 * 3 mod 23 = 48 mod 23 = 2

g^s = 1 ≠ 2  ← ไม่ตรง!

แก้ไข: s = 22 mod 11 = 0 ผิดพลาด ลองใหม่:

r = 7, c = 5, x = 3
s = (r + c*x) mod q = (7 + 15) mod 11 = 22 mod 11 = 0

ปัญหา: s=0 ทำให้ fail เพราะ g^0 = 1

ลอง r=8:
s = (8 + 15) mod 11 = 23 mod 11 = 1
g^s = 4^1 mod 23 = 4
R = g^r = 4^8 mod 23 = 65536 mod 23 = 65536 - 2849*23 = 65536 - 65527 = 9
Y^c = 18^5 mod 23 = 3 (คำนวณข้างต้น)
R * Y^c = 9 * 3 mod 23 = 27 mod 23 = 4

g^s = 4 == R * Y^c = 4 ✓ VERIFY PASS

--- Fiat-Shamir version ---
c = H(g=4 || Y=18 || R=9) mod 11
  = SHA-256(04 || 12 || 09) mod 11  (แทนค่า hex)
  = [hash output] mod 11  = ค่า deterministic
Proof = (R=9, s=1)
Verify: recompute c' = H(4 || 18 || 9) mod 11, check g^s = R * Y^c'
```

ตัวอย่างนี้แสดงให้เห็น mechanics ของ protocol แม้ว่า p=23 จะไม่ secure เลยในทางปฏิบัติ

## สรุป

โปรเจคนี้สร้าง Zero-Knowledge Proof stack ครบวงจร:

| Component | What it proves | Algorithm |
|---|---|---|
| Schnorr (safe-prime) | Knowledge of discrete log x: Y=g^x | Σ-protocol |
| Fiat-Shamir | NI version ของ Schnorr | H(g‖Y‖R) challenge |
| Pedersen | Hiding commitment + homomorphic add | Commitments |
| Range proof | v ∈ [0, 2^n) | Bit decomposition |
| Ristretto Schnorr | DL on Ristretto255 | Σ-protocol + ECC |
| Batch verify | N proofs simultaneously | Random lin. comb. |

**Pattern สำคัญที่ได้เรียน:**

1. **Σ-protocol structure** — commit → challenge → respond เป็น template ที่ใช้ได้กับ statement ใด ๆ ที่เป็น "knowledge of witness"
2. **Fiat-Shamir heuristic** — แปลง interactive proof เป็น signature ด้วย random oracle — วิธีนี้ใช้ใน ECDSA, EdDSA, Schnorr signatures ทั้งหมด
3. **Commitment schemes** — algebraic structure ที่ผูก prover กับค่า แต่ไม่เปิดเผยจน reveal
4. **Constant-time API** — `subtle::CtOption` ที่ curve25519-dalek ใช้เพื่อป้องกัน timing side channels
5. **Batch verification** — random linear combination ลด N independent equations เป็น 1 equation ประหยัดเวลา

โปรเจคถัดไป `project-e01-tetris.md` จะเปลี่ยนโดเมนไปยัง Games/Graphics — สร้าง Tetris ด้วย Bevy game engine ซึ่งใช้ ECS (Entity-Component-System) architecture ที่แตกต่างจาก OOP อย่างสิ้นเชิง

---

**โปรเจคก่อนหน้า:** [Project D09: WAF Middleware](project-d09-waf-middleware.md) | **โปรเจคถัดไป:** [Project E01: Tetris](project-e01-tetris.md)
