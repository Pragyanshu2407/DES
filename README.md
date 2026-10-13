# DES demo — presentation kit

| File | What it is |
|---|---|
| `des_demo.html` | Interactive step-by-step DES explorer. Double-click to open — no install, works offline. |
| `des.py` | ~100-line from-scratch DES in Python (`python3 des.py` runs a self-test). |

**Controls:** `→` / `Space` = next, `←` = back, Auto-play button, click any step tab or round number.

## Test vector (use this live)
- Key: `133457799BBCDFF1`
- Plaintext: `0123456789ABCDEF`
- Ciphertext: `85E813540F0AB405`

## Suggested talk track (~8 min)
1. **Input** – 64-bit block + 64-bit key; every 8th key bit is parity, so only 56 bits matter.
2. **Initial Permutation** – a fixed shuffle (no security value); then split into L0 / R0.
3. **Key schedule** – one key -> sixteen 48-bit round keys via PC-1, rotations, PC-2.
4. **Rounds** – the heart: `L <- R`, `R <- L xor f(R, K)`. Use *Next* to reveal Expand -> xor key -> S-boxes -> P-box -> xor & swap. Click an S-box to show its table lookup (outer bits = row, inner 4 = column).
5. **Output** – swap + inverse permutation = ciphertext. Decrypt = same circuit, keys reversed (green "round-trip verified" badge, or press *Decrypt*).
6. **Avalanche** – choose *where* to flip one bit: the input, the state entering any round 1–16, or a key bit. The chart and the round-by-round table (changed bits in red) show rounds before the flip are identical and the difference spreads to ~32/64 bits. Flip a parity key bit to show *nothing* changes.

## Closing points
- 56-bit key -> brute-forced (EFF Deep Crack, 1998, < 3 days).
- 3DES stretched its life; AES replaced it in 2001.
