# Code Explained — how the DES demo works

This project has two implementations of the same algorithm:

| File | Language | Purpose |
|---|---|---|
| `des.py` | Python | Short, readable reference implementation (and test) |
| `des_demo.html` | JavaScript + HTML/CSS | Interactive visual demo (same algorithm, plus a trace for the UI) |

Both produce identical output (checked on random inputs). Read `des.py` first — it is the simplest view of DES.

---

## 1. The big idea in one picture

```
 64-bit plaintext ──► IP ──► split L0 | R0
                              │
        repeat 16 times:      │   L(i) = R(i-1)
                              │   R(i) = L(i-1) XOR f( R(i-1), K(i) )
                              ▼
                      swap halves ──► FP ──► 64-bit ciphertext
```

- **Block cipher:** always works on exactly 64 bits.
- **Feistel network:** each round only *XORs* one half with a function of the other half. Because XOR is its own inverse, `f` never has to be reversible.
- **Decryption = encryption with the 16 round keys in reverse order.** No separate decrypt algorithm.

---

## 2. Everything is a list of bits

Both files represent data as a list of `0`/`1` (not as integers). This is slower than real DES code but makes every step printable and easy to explain.

### Helper functions (same idea in both files)

| Python | JavaScript | What it does |
|---|---|---|
| `permute(bits, table)` | `permute(b, t)` | Builds a new list: output position *i* = input bit `table[i]`. Tables are **1-indexed**, hence `bits[i-1]`. Used for IP, FP, E, P, PC-1, PC-2. |
| `xor(a, b)` | `xor(a, b)` | Bitwise XOR of two equal-length lists. |
| `rotl(bits, n)` | `rot(a, n)` | Circular left shift: `bits[n:] + bits[:n]`. |
| `hex_to_bits` / `bits_to_hex` | `hexToBits` / `bitsToHex` | Convert between a hex string and a bit list. |

---

## 3. The tables (constants)

| Table | Size | Role |
|---|---|---|
| `IP` | 64 → 64 | Initial permutation (fixed shuffle at the start) |
| `FP` | 64 → 64 | Final permutation = inverse of IP |
| `E` | 32 → 48 | Expansion: duplicates edge bits so 32 bits become 48 |
| `P` | 32 → 32 | Permutation applied after the S-boxes |
| `PC1` | 64 → 56 | Key permutation; drops the 8 parity bits |
| `PC2` | 56 → 48 | Picks 48 bits of the rotated key for each round |
| `SHIFTS` | 16 values | How many bits to rotate in each round: `1,1,2,2,2,2,2,2,1,2,2,2,2,2,2,1` |
| `SBOXES` / `T.S` | 8 boxes × 64 | The substitution tables — the only **non-linear** part of DES |

These are the published standard values; changing even one number breaks the test vector.

---

## 4. Key schedule — `make_subkeys` (Python) / `keySchedule` (JS)

```python
k = permute(key_bits, PC1)        # 64 -> 56 bits (parity bits dropped)
c, d = k[:28], k[28:]             # two 28-bit halves
for s in SHIFTS:
    c, d = rotl(c, s), rotl(d, s) # rotate both halves left by 1 or 2
    keys.append(permute(c + d, PC2))  # choose 48 bits -> round key
```

Result: **16 round keys of 48 bits**. Note `c` and `d` keep rotating cumulatively — round 2 starts from round 1's rotated values.

In JS the same function also returns every intermediate `C`/`D` so the UI can show them in the Key Schedule table.

---

## 5. The round function `f` — `feistel` (Python)

```python
x = permute(r, E)       # 1. Expand   32 -> 48 bits
x = xor(x, subkey)      # 2. Mix key  XOR with the 48-bit round key
x = sbox_layer(x)       # 3. Substitute 48 -> 32 bits (8 S-boxes)
return permute(x, P)    # 4. Permute
```

### The S-box step (`sbox_layer`)
The 48 bits are cut into eight 6-bit chunks. For each chunk `b0 b1 b2 b3 b4 b5`:

- **row** = outer bits `b0 b5` (2 bits → 0..3)
- **column** = middle bits `b1 b2 b3 b4` (4 bits → 0..15)
- look up `SBOXES[i][row*16 + col]` → a number 0..15 → write as 4 bits

So 8 × 6 bits in → 8 × 4 bits out = 32 bits. This is the step the demo highlights with the row/column colours.

---

## 6. Whole block — `des_block` (Python) / `run` (JS)

```python
b = permute(block_bits, IP)
l, r = b[:32], b[32:]
for k in subkeys:
    l, r = r, xor(l, feistel(r, k))   # one Feistel round
return permute(r + l, FP)              # NOTE: r + l  (halves swapped)
```

Two details students often miss:
1. After round 16 the halves are **swapped** (`r + l`) before FP.
2. For decryption, the *only* change is passing `subkeys[::-1]` (Python) or `order = rows.reverse()` (JS).

`encrypt()` / `decrypt()` in `des.py` just wrap this with hex conversion.

---

## 7. How the HTML demo is organised

`des_demo.html` is one file with three parts.

### a) The engine (between `==ENGINE-START==` and `==ENGINE-END==`)
Pure logic, no HTML. Key functions:

- `keySchedule(keyBits)` → `{ c0, d0, rows[] }` with each round's `C`, `D`, shift and 48-bit key.
- `run(blockBits, keyBits, decrypt, inj)` → a **trace object**:
  - `ip` — block after IP
  - `rounds[]` — for each round: `L`, `R`, `exp` (expanded), `key`, `x` (after XOR), `sb[]` (each S-box's chunk/row/col/value), `sOut`, `p`, `newL`, `newR`
  - `pre` — halves swapped, `out` — final ciphertext bits
  - `inj` (optional) flips one bit of the 64-bit state at the start of round `inj.round` — this powers the "inject at any round" avalanche feature.

Because `run` records *everything*, the UI never recomputes DES; it only draws the trace.

### b) State and rendering
A single object `S` holds the UI state: current tab, round, sub-step, mode (enc/dec), avalanche settings, and the latest trace `S.tr`.

| Function | Draws |
|---|---|
| `vInput`, `vIP`, `vKey` | Tabs 1–3 |
| `vRound` | Tab 4 — six sub-sections (split / expand / key mix / S-boxes / P-box / XOR & swap) plus the SVG Feistel diagram |
| `vOutput` | Tab 5 — final permutation, ciphertext, round-trip check (re-runs the opposite direction and compares) |
| `vAv` | Tab 6 — avalanche: runs DES twice (original vs. one bit flipped) and diffs the states round by round |
| `render()` | Updates tab bar, progress bar, and calls the right `v…` function |

Small helpers: `bits()` / `bit()` draw coloured 0/1 digits, `grid()` draws the 8×8 bit grids, `hex()` prints hex.

### c) Events
- `compute()` runs on every input change: parses plaintext/key → builds the trace → `render()`.
- `parse()` accepts 16 hex digits **or** up to 8 text characters (zero-padded to 8 bytes).
- `next()` / `prev()` walk through steps; inside the Rounds tab they step through the 6 sub-sections, then through rounds 1–16.
- Keyboard: `→`/`Space` next, `←` back.

---

## 8. Avalanche feature in detail

1. Pick the injection point `r` (0 = input, 1–16 = state entering that round, or key).
2. Run DES normally → trace A.
3. Flip the chosen bit and run again → trace B (`run(..., {round, bit})` for round injection).
4. For every stage count `diff(A, B)` and highlight differing bits in red.

Before the injection round the two traces are identical (difference = 0). After it, the difference grows to roughly 32 of 64 bits within a few rounds — that is the avalanche effect.

---

## 9. Verification

Standard test vector:

| | Value |
|---|---|
| Key | `133457799BBCDFF1` |
| Plaintext | `0123456789ABCDEF` |
| Ciphertext | `85E813540F0AB405` |

- `python3 des.py` asserts encryption matches this and that decryption restores the plaintext.
- The JS engine was cross-checked against the Python version on random key/plaintext pairs (encrypt and decrypt).

---

## 10. Limitations (good to mention in the talk)

- Single 64-bit block only (ECB); no padding scheme, no chaining modes.
- Bit-list representation is for clarity, not speed or security.
- Not constant-time and not for protecting real data — DES itself is obsolete (56-bit key, brute-forceable). Use AES.
