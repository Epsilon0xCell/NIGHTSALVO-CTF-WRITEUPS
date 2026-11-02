# Intercepted Transmissions

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** beginner
- **Author:** Pix

## Summary

The handout file `publish/encoding` is a single string of 441 bits: a CCIR476 transmission from the fictional kidnappers. The challenge states that the first two decoded characters are `##` and that the decoded message, wrapped in `DUCTF{}`, is the flag.

## Solve

CCIR476 is a 7-bit constant-ratio code: every character is a 7-bit group containing exactly four mark bits. Grouping the stream into 7-bit codewords from offset 0 gives 63 codewords, and every one of them has weight exactly 4, which confirms the grouping, the bit order (most significant bit first), and the polarity in one check. There are only 21 distinct codewords, well inside the 35 valid CCIR476 code points.

CCIR476 is stateful like the ITA2/Baudot alphabet it recasts: codeword `0x5A` is LTRS (switch to the letters set), `0x36` is FIGS (switch to the figures set), and `0x5C` is space in both sets. Every other codeword is looked up in whichever set is active. Key mappings used: `0x69` is H in letters and `#` in figures (the US figures variant, confirmed by the known `##` start), `0x56` is E/3, `0x2D` is P/0, `0x55` is R/4, `0x4B` is S/', `0x2E` is Q/1, `0x1B` is F/!, and `0x71` is O/9.

```python
bits = open('publish/encoding').read().strip()
groups = [bits[i:i+7] for i in range(0, len(bits), 7)]
assert all(g.count('1') == 4 for g in groups)   # constant ratio holds

LTRS, FIGS, SPACE = 0x5A, 0x36, 0x5C
# letters / figures tables keyed by codeword value (CCIR476 chart)
out, mode = [], 'L'
for g in groups:
    v = int(g, 2)
    if v == LTRS:
        mode = 'L'
    elif v == FIGS:
        mode = 'F'
    elif v == SPACE:
        out.append(' ')
    else:
        out.append(TABLE[mode][v])
message = ''.join(out)
```

The stream opens with LTRS, FIGS, `0x69`, `0x69`, which decodes to the known first two characters `##` and validates the whole table and bit order. The frequent LTRS/FIGS shifts in the stream are genuine: the message is written in leetspeak, mixing letters with the figures 3, 0, 4 and 1, so the encoder had to shift sets constantly.

Decoded message:

```
##TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!
```

Verification by re-encoding: feeding the decoded message back through a standard CCIR476 encoder (emit LTRS/FIGS on every set change, space = `0x5C`) reproduces the handout bit stream exactly, codeword for codeword, all 63 groups including the leading LTRS. The decode is exact, with no guessed characters.

## Flag

```
DUCTF{##TH3 QU0KK4'S AR3 H3LD 1N F4C1LITY #11911!}
```
