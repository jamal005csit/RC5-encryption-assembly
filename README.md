# RC5 Block Cipher — ATmega32 AVR Assembly Implementation

An implementation of the RC5-16/8/12 symmetric block cipher written in AVR assembly for the ATmega32 microcontroller. The cipher operates on 16-bit word pairs, uses an 8-round Feistel-like structure, and derives its key schedule from a 12-byte secret key.

---

## Table of Contents

- [Algorithm Overview](#algorithm-overview)
- [Parameters](#parameters)
- [Memory Layout](#memory-layout)
- [Project Structure](#project-structure)
- [Subroutine Reference](#subroutine-reference)
- [Register Conventions](#register-conventions)
- [Key Expansion](#key-expansion)
- [Encryption](#encryption)
- [Decryption](#decryption)
- [How to Build and Run](#how-to-build-and-run)
- [Example](#example)

---

## Algorithm Overview

RC5 is a parameterized, symmetric-key block cipher designed by Ronald Rivest in 1994. This implementation uses the following variant:

```
RC5 - W/R/B
      16 / 8 / 12
      │    │   └── Key length in bytes (12 bytes = 96 bits)
      │    └────── Number of rounds (8)
      └─────────── Word size in bits (16-bit words)
```

The cipher takes a 32-bit plaintext block (two 16-bit words, A and B) and a 12-byte key, expands the key into 18 subkeys, then applies 8 rounds of mixing using XOR, modular addition, and variable rotation.

---

## Parameters

| Constant  | Value    | Description                                |
|-----------|----------|--------------------------------------------|
| `P16`     | `0xB7E1` | Magic constant derived from (e - 2) × 2^16 |
| `Q16`     | `0x9E37` | Magic constant derived from (φ - 1) × 2^16 |
| `ROUNDS`  | `8`      | Number of encryption/decryption rounds     |
| `T`       | `18`     | Number of subkeys (= 2 × ROUNDS + 2)       |
| `C`       | `6`      | Number of 16-bit words in the original key |

---

## Memory Layout

### Data Segment (SRAM, starting at 0x60)

| Symbol | Size     | Description                                      |
|--------|----------|--------------------------------------------------|
| `S`    | 36 bytes | Expanded key schedule — 18 × 16-bit subkeys      |
| `L`    | 12 bytes | Temporary key array — 6 × 16-bit words from K    |

### Program Segment (Flash)

| Symbol   | Location | Description                          |
|----------|----------|--------------------------------------|
| `K_data` | Flash    | 12-byte secret key stored as `.db`   |

---

## Project Structure

```
rc5_avr.asm
│
├── K_data          — Secret key bytes (stored in Flash)
│
├── RotL16          — 16-bit circular left rotate subroutine
├── RotR16          — 16-bit circular right rotate subroutine
│
├── KeyExpansion    — Derives 18 subkeys from the secret key
│
├── Encrypt         — RC5 encryption of a 32-bit block
├── Decrypt         — RC5 decryption of a 32-bit block
│
└── MAIN            — Entry point: sets up stack, runs demo
```

---

## Subroutine Reference

### `RotL16`

Performs a 16-bit circular left rotation.

- **Input:** `r25:r24` — value to rotate | `r23` — number of positions (masked to 0–15)
- **Output:** `r25:r24` — rotated result
- **Flags used:** C (Carry) — used as the bridge between the two bytes
- **Method:** `lsl r24` pushes bit 7 into Carry; `rol r25` pulls Carry into bit 0 of r25. If Carry is still set after `rol`, bit 15 wraps back into bit 0 of r24 via `ori r24, 0x01`.

```asm
lsl  r24        ; bit7 of r24 → Carry; 0 enters bit0
rol  r25        ; Carry → bit0 of r25; bit7 of r25 → Carry
brcc skip       ; if Carry = 0, no wrap needed
ori  r24, 0x01  ; wrap bit15 back into bit0
```

---

### `RotR16`

Performs a 16-bit circular right rotation.

- **Input:** `r25:r24` — value to rotate | `r23` — number of positions (masked to 0–15)
- **Output:** `r25:r24` — rotated result
- **Flags used:** T (Transfer) — used to save bit 0 before shifting; C (Carry) — used by the shift chain
- **Method:** bit 0 of r24 is saved into T first (via `bst`). Then `lsr r25` + `ror r24` shifts the full 16-bit value right. If T was set, bit 0 wraps into bit 15 of r25 via `ori r25, 0x80`.

```asm
bst  r24, 0     ; save bit0 of r24 into T flag
lsr  r25        ; shift r25 right; bit0 → Carry; 0 → bit7
ror  r24        ; Carry → bit7 of r24; bit0 of r24 → Carry
brtc skip       ; if T = 0, no wrap needed
ori  r25, 0x80  ; wrap saved bit0 into bit15
```

> **Why T and not Carry?** `lsr r25` would overwrite Carry before we can use it. T is independent of shift instructions, making it the safe choice.

---

### `KeyExpansion`

Derives the 18-element subkey array `S[]` from the raw 12-byte key `K_data`.

**Steps:**

1. **Initialize L[]** — Copy `K_data` from Flash into `L` in SRAM, interpreting every two bytes as a little-endian 16-bit word.
2. **Initialize S[]** — Set `S[0] = P16`, then `S[i] = S[i-1] + Q16` for i = 1..17.
3. **Mix** — Iterate 54 times (= 3 × max(T, C)):
   - Update `S[i mod 18]` using the current A and B accumulator values.
   - Update `L[j mod 6]` using the new S value.
   - Increment both indices independently.

---

### `Encrypt`

Encrypts a 32-bit plaintext block using the expanded key schedule.

- **Input:** `r17:r16` = A (low word) | `r19:r18` = B (high word)
- **Output:** `r17:r16` = encrypted A | `r19:r18` = encrypted B
- **S[] must be populated** by `KeyExpansion` before calling.

**Algorithm:**

```
A = A + S[0]
B = B + S[1]
for i = 1 to 8:
    A = ROL(A XOR B, B) + S[2i]
    B = ROL(B XOR A, A) + S[2i+1]
```

---

### `Decrypt`

Decrypts a 32-bit ciphertext block — exact inverse of `Encrypt`.

- **Input:** `r17:r16` = A (encrypted) | `r19:r18` = B (encrypted)
- **Output:** `r17:r16` = original A | `r19:r18` = original B

**Algorithm:**

```
for i = 8 downto 1:
    B = ROR(B - S[2i+1], A) XOR A
    A = ROR(A - S[2i],   B) XOR B
A = A - S[0]
B = B - S[1]
```

> **Note:** B is processed before A in each round — the reverse of Encrypt, which updates A first.

---

## Register Conventions

| Register(s)  | Role                                           |
|--------------|------------------------------------------------|
| `r17:r16`    | A — low 16-bit word of the plaintext/ciphertext|
| `r19:r18`    | B — high 16-bit word                           |
| `r25:r24`    | Temporary value passed to/from rotate routines |
| `r23`        | Rotation count for RotL16 / RotR16             |
| `r20`        | Loop counter                                   |
| `r18`, `r19` | Indices i and j in KeyExpansion mix loop       |
| `Z (r31:r30)`| Pointer into S[] (SRAM or Flash)               |
| `X (r27:r26)`| Pointer into L[] (SRAM)                        |

---

## How to Build and Run

### Requirements

- [AVR-GCC / AVR Assembler](https://www.microchip.com/en-us/tools-resources/develop/microchip-studio) or [AVRA](https://github.com/Ro5bert/avra)
- [AVR simulator](https://www.microchip.com/en-us/tools-resources/develop/microchip-studio) (Microchip Studio / Atmel Studio) or physical ATmega32 with a programmer

### Assemble

```bash
avra rc5_avr.asm
```

### Simulate

Open `rc5_avr.hex` in Microchip Studio's simulator. Set a breakpoint at `FINISH` and inspect the register file to see the encrypted and decrypted values in `r17:r16` and `r19:r18`.

---

## Example

The `MAIN` routine demonstrates a full encrypt → decrypt cycle:

```asm
; Plaintext loaded into registers:
ldi r16, 0x34   ; A low byte
ldi r17, 0x12   ; A high byte  →  A = 0x1234
ldi r18, 0x78   ; B low byte
ldi r19, 0x56   ; B high byte  →  B = 0x5678

call Encrypt     ; A:B now holds ciphertext
call Decrypt     ; A:B restored to 0x1234 / 0x5678
```

After `Decrypt` completes, `r17:r16` should return to `0x1234` and `r19:r18` to `0x5678`, confirming correctness of both routines.

---

## Notes

- All arithmetic is **modular 2^16** (16-bit wrap-around), which is the natural behavior of AVR 16-bit add/subtract pairs.
- The `lpm` instruction is used to read `K_data` from Flash. Flash addresses in AVR are word-addressed, so the pointer is computed as `K_data << 1`.
- Rotation by more than 15 positions on a 16-bit word is equivalent to a smaller rotation. The `andi r23, 0x0F` mask enforces this and prevents runaway loops.
- This implementation targets the **ATmega32** (`m32def.inc`). Porting to other AVR devices requires updating the include file and verifying SRAM address compatibility.
