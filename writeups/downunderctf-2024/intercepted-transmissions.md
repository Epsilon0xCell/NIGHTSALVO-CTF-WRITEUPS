# Intercepted Transmissions

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** beginner
- **Author:** Pix

## Summary

The handout file `publish/encoding` is a single string of 441 bits: a CCIR476 transmission from the fictional kidnappers. The challenge states that the first two decoded characters are `##` and that the decoded message, wrapped in `DUCTF{}`, is the flag.

## First look

```
$ ls -la publish/
total 4
drwxrwx--- 1 root root  16 Nov  2 08:45 .
drwxrwx--- 1 root root  56 Nov  2 09:18 ..
-rw-rw---- 1 root root 442 Nov  2 08:45 encoding
$ head -c 200 publish/encoding; echo
10110100110110110100111010011011010111010011010010110110101011010111001011010010111010011100110110010110110110100011110001111001101101010101100101110110101001011101110010001111010101011011010101101011
$ wc -c publish/encoding
442 publish/encoding
```

442 bytes on disk is 441 bits plus a trailing newline. The README names the encoding (CCIR476), gives the first two decoded characters as `##`, and says to wrap the decoded message in `DUCTF{}`. Everything else has to come out of the bit stream.

## What I tried first

**Is it plain Baudot?** CCIR476 is a recasting of the ITA2/Baudot alphabet, and Baudot is 5 bits per character, so the first check is whether the stream even groups into 5s:

```
441 % 5 = 1 -> not a 5-bit code, length rules Baudot out
```

441 is not divisible by 5. Dead on arrival, and a good reminder to check the arithmetic before the alphabet.

**Grouping into 7s, but from the wrong offset.** CCIR476 is a 7-bit constant-ratio code: every character is a 7-bit group containing exactly four mark bits. That property is strong enough to test alignment before decoding anything:

```
offset 0: 63 groups, 0 groups without exactly four 1 bits
offset 1: 62 groups, 27 groups without exactly four 1 bits
```

Offset 0 is perfect, 63 groups and every one of weight exactly 4. Offset 1 breaks the property in 27 of 62 groups. So the grouping, the bit order (most significant bit first), and the polarity are all confirmed as written, in one test, before a single character is decoded. There are only 21 distinct codewords in the stream, well inside the 35 valid CCIR476 code points.

**Decoding without the shift state.** CCIR476 is stateful like the ITA2 alphabet it recasts: codeword `0x5A` is LTRS (switch to the letters set), `0x36` is FIGS (switch to the figures set), `0x5C` is space in both sets, and every other codeword means one thing in letters and another in figures. My first decode ignored the shifts and read everything as letters:

```
letters-only decode starts: HHTHE QUPKKR -> contradicts the known '##' start, the shifts are real
```

`HH` where the challenge promises `##`, and the rest is near-English gibberish. That failure is informative: `0x69` is `H` in letters and `#` in figures (the US figures variant), so the known `##` start is only reachable if the decoder honours the FIGS shift that precedes it. The stream opens LTRS, FIGS, `0x69`, `0x69`, which is a shift into figures and then the two promised `#` characters.

## The breakthrough

The constant-ratio check plus the known plaintext. Once offset 0 grouping passed 63 for 63 and the state machine's first two output characters matched the given `##`, the rest of the decode had a built-in referee: the output either reads as a message or it does not. The frequent LTRS/FIGS shifts in the stream turn out to be genuine, not noise: the message is written in leetspeak, mixing letters with the figures 3, 0, 4 and 1, so the encoder had to shift sets constantly. Key mappings on the path: `0x69` is H/#, `0x56` is E/3, `0x2D` is P/0, `0x55` is R/4, `0x4B` is S/', `0x2E` is Q/1, `0x1B` is F/!, `0x71` is O/9.

## Solve

```python
bits = open('publish/encoding').read().strip()
print("bits:", len(bits))

# Dead end 0: is it plain 5-bit Baudot/ITA2? 441 % 5 != 0, so no.
print("441 % 5 =", len(bits) % 5, "-> not a 5-bit code, length rules Baudot out")

# CCIR476 is 7-bit. Check the constant-ratio property at both offsets.
for off in (0, 1):
    gs = [bits[i:i+7] for i in range(off, len(bits) - 6, 7)]
    bad = sum(1 for g in gs if g.count('1') != 4)
    print(f"offset {off}: {len(gs)} groups, {bad} groups without exactly four 1 bits")
groups = [int(bits[i:i+7], 2) for i in range(0, len(bits), 7)]
print("offset 0 it is:", len(groups), "codewords,",
      len(set(groups)), "distinct")

LTRS = {0x47: 'A', 0x1D: 'C', 0x53: 'D', 0x56: 'E', 0x1B: 'F', 0x69: 'H',
        0x4D: 'I', 0x1E: 'K', 0x65: 'L', 0x59: 'N', 0x71: 'O', 0x2D: 'P',
        0x2E: 'Q', 0x55: 'R', 0x4B: 'S', 0x74: 'T', 0x4E: 'U', 0x2B: 'Y'}
FIGS = {0x56: '3', 0x2D: '0', 0x55: '4', 0x4B: "'", 0x2E: '1', 0x1B: '!',
        0x71: '9', 0x69: '#'}

# Dead end 1: ignore the LTRS/FIGS shifts, decode everything as letters.
naive = ''.join(' ' if v == 0x5C else LTRS.get(v, '?')
                for v in groups if v not in (0x5A, 0x36))
print("letters-only decode starts:", naive[:12],
      "-> contradicts the known '##' start, the shifts are real")

# Real decode: state machine over letters/figures sets.
out, mode = [], 'L'
for v in groups:
    if v == 0x5A:
        mode = 'L'
    elif v == 0x36:
        mode = 'F'
    elif v == 0x5C:
        out.append(' ')
    else:
        out.append((LTRS if mode == 'L' else FIGS)[v])
msg = ''.join(out)
print("decoded:", msg)
print("starts with the known '##':", msg.startswith('##'))

# Verification: re-encode and compare bit for bit. The handout carries one
# redundant leading LTRS (a sync shift) before the first FIGS, so the
# re-encode starts with it too.
REV_L = {c: v for v, c in LTRS.items()}
REV_F = {c: v for v, c in FIGS.items()}
enc, mode = [0x5A], 'L'
for ch in msg:
    if ch == ' ':
        enc.append(0x5C)
        continue
    if ch in REV_L and (mode == 'L' or ch not in REV_F):
        if mode != 'L':
            enc.append(0x5A); mode = 'L'
        enc.append(REV_L[ch])
    else:
        if mode != 'F':
            enc.append(0x36); mode = 'F'
        enc.append(REV_F[ch])
re_bits = ''.join(f'{v:07b}' for v in enc)
print("re-encoded stream matches handout exactly:", re_bits == bits)
print("flag: DUCTF{" + msg + "}")
```

One honest note on the verification step: my first re-encode did **not** match (`re-encoded stream matches handout exactly: False`, 62 codewords against the handout's 63). Diffing the two streams codeword by codeword showed a single extra `0x5A` at the very front of the handout: a redundant LTRS sync shift before the first FIGS, which changes nothing in the decode (the decoder starts in letters anyway) but has to be reproduced for a bit-exact re-encode. With that one shift accounted for, the match is exact.

Run:

```
$ python3 intercepted_solve.py
bits: 441
441 % 5 = 1 -> not a 5-bit code, length rules Baudot out
offset 0: 63 groups, 0 groups without exactly four 1 bits
offset 1: 62 groups, 27 groups without exactly four 1 bits
offset 0 it is: 63 codewords, 21 distinct
letters-only decode starts: HHTHE QUPKKR -> contradicts the known '##' start, the shifts are real
decoded: ##TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!
starts with the known '##': True
re-encoded stream matches handout exactly: True
flag: DUCTF{##TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!}
```

## Flag

```
DUCTF{##TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!}
```

Sanity check: the decoded message starts with the two characters the challenge gives away (`##`), reads as a complete leetspeak sentence (`TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!`), and re-encoding it reproduces the handout bit stream exactly, all 63 codewords, so no character in the flag rests on a guess.
