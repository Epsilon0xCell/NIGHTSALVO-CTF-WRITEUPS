# Flag Art

- **Event:** DownUnderCTF 2023
- **Category:** beginner (misc/crypto)
- **Source:** official DownUnderCTF/Challenges_2023_Public repo, `beginner/flag-art`
- **Handouts:** `flag-art.py`, `output.txt`

## Summary

The generator turns a message (a padding text with the flag appended) into ASCII art. For every byte it emits four symbols from a seven character palette, encoding the byte's residues modulo 2, 3, 5, and 7. Those four residues pin the byte down modulo 210, and every byte of an ASCII message is below 128, so the Chinese Remainder Theorem rebuilds each character exactly. I extracted the 900 emitted symbols from the artwork in reading order, grouped them in fours, and CRT-decoded all 225 bytes.

## First look

The generator logic:

```python
message = open('./message.txt', 'rb').read() + open('./flag.txt', 'rb').read()

palette = '.=w-o^*'
template = list(open('./mask.txt', 'r').read())

canvas = ''
for c in message:
    for m in [2, 3, 5, 7]:
        while True:
            t = template.pop(0)
            if t == 'X':
                canvas += palette[c % m]
                break
            else:
                canvas += t

print(canvas)
```

The artwork itself is a picture drawn in spaces and palette characters. The mask template is not in the handouts, but it does not need to be: every non-X template character is copied into the canvas verbatim, and in this artwork the background characters are spaces. The palette characters in the canvas are exactly the emissions, in order.

## What I tried first

Two wrong turns, both mine. First, I wondered whether template characters could collide with the palette (a literal '.' or 'w' drawn by the mask would corrupt the extraction). Counting settled it: the canvas holds exactly 900 palette characters, and 900 = 4 x 225, a whole number of bytes with nothing left over. If mask characters were polluting the stream, that arithmetic would be very unlikely to come out clean, and the decoded text later confirmed it.

Second, I grouped the residues in the wrong modulus order, [7, 5, 3, 2] instead of the generator's [2, 3, 5, 7]. CRT is order-sensitive in the sense that each residue belongs to its own modulus; scrambling the pairing produced bytes like b'\xb0\xbe\x93\x8dF\xb0p\x9bi\xb0'. That garbage was useful: it proved the extraction was right and only the pairing was wrong, because the values were structured rather than random.

## The breakthrough

Reading the generator's loop again: for byte c and modulus m in [2, 3, 5, 7], it emits palette[c % m], and the palette index is the residue itself (indices 0 to 6 cover all residues that 2, 3, 5, and 7 can produce). So each group of four symbols is the residue vector of one byte, and CRT over moduli 2, 3, 5, 7 gives the byte modulo 210. Text bytes are all below 128, so there is exactly one candidate per byte. No search, no ambiguity.

## Solve

```python
canvas = open('../handouts/flag-art-output.txt').read()
palette = '.=w-o^*'
syms = [palette.index(ch) for ch in canvas if ch in palette]
print('emitted symbols:', len(syms), '-> bytes:', len(syms) // 4)

def crt(res, mods):
    x, M = 0, 1
    for r, m in zip(res, mods):
        t = ((r - x) * pow(M, -1, m)) % m
        x += M * t
        M *= m
    return x % M

msg = bytes(crt(syms[i:i + 4], [2, 3, 5, 7]) for i in range(0, len(syms) - 3, 4))
print(msg.decode())
```

Output:

```
emitted symbols: 900 -> bytes: 225
Congratulations on solving this challenge! The mask has 900 X's so here are some random words to make the message long enough. Your flag is: DUCTF{r3c0nstruct10n_0f_fl4g_fr0m_fl4g_4r7_by_l00kup_t4bl3_0r_ch1n3s3_r3m41nd3r1ng?}
```

Most of the 225 bytes are the challenge padding its own message with filler words so the artwork has room to exist. The flag rides at the end, as the generator's message-plus-flag construction says it should.

## Flag

`DUCTF{r3c0nstruct10n_0f_fl4g_fr0m_fl4g_4r7_by_l00kup_t4bl3_0r_ch1n3s3_r3m41nd3r1ng?}`
