# Hex Shift Cipher

- **Event:** DownUnderCTF 2020
- **Category:** crypto
- **Difficulty:** medium
- **Author:** joseph#8210

## Summary

"People say shift ciphers aren't secure. I'm here to prove them wrong!" The handout is the cipher source and its output. A secret message is hex-encoded behind the known prefix "The secret message is:", then encrypted with a cipher built on a shuffled hex alphabet and a nibble of feedback. The proving-wrong does not survive contact with linear algebra over GF(2).

## First look

```python
ALPHABET = '0123456789abcdef'

class Cipher:
    def __init__(self, key):
        self.key = key          # a shuffled copy of ALPHABET
        self.n = len(self.key)
        self.s = 7

    def add(self, num1, num2):  # bitwise: res bit i = (num1 bit i + num2 bit i) % 2
        ...

    def encrypt(self, msg):
        key = self.key
        s = self.s
        ciphertext = ''
        for m_i in msg:
            c_i = key[self.add(key.index(m_i), s)]
            ciphertext += c_i
            s = key.index(m_i)
        return ciphertext

plaintext = b'The secret message is:'.hex() + secret_msg.hex()
```

Write `pos(x)` for `key.index(x)`, the position of hex digit `x` in the shuffled alphabet. The `add` function is a per-bit sum mod 2, which is XOR, so encryption is `pos(c_i) = pos(m_i) XOR s_i`, with `s_0 = 7` and the feedback `s_{i+1} = pos(m_i)`. Everything unknown about this cipher lives in `pos`, a bijection from the 16 hex digits to the values 0 to 15. Sixteen unknowns, four bits each, and every known plaintext character produces one equation: for `i >= 1`, `pos(m_i) XOR pos(m_{i-1}) XOR pos(c_i) = 0`, and for the first character, `pos(m_0) XOR pos(c_0) = 7`. The known prefix is 44 hex characters long, so the handout itself supplies 44 equations.

## What I tried first

**Treating the equations as a set system.** My first build collected each equation's variables in a Python `set`, and the solver promptly reported the system inconsistent, with no free variables to blame. The bug is instructive: when the same digit appears twice in one equation (say `m_i == m_{i-1}`), its `pos` cancels under XOR, but a set silently keeps one copy and poisons the equation. The transcription of the ciphertext was the other suspect, so I checked that first: the string in my solver matches the source comment byte for byte, all 264 characters. With the transcription cleared, the set multiplicity was the only suspect left, and toggling variables in and out of a mask instead of unioning a set fixed it immediately.

**Brute force over the key.** Sixteen factorial is about 2 x 10^13 shuffled alphabets, far too many to enumerate, and the feedback makes partial-key scoring awkward. It was never needed: the equations decide almost everything.

## The breakthrough

The system is linear over GF(2) in each bit independently, so one Gaussian elimination over 16 variables handles all four bits at once if the right-hand sides ride along as a packed nibble. Eliminating the 44 equations leaves three free variables, the positions of the digits d, e and f, which the known prefix never pins down linearly: 4,096 raw completions. What the linear solve cannot see is bijectivity, `pos` has to be a permutation, so I enumerate the free variables and keep the 1,344 assignments that are genuine permutations. Every single one decrypts to the same plaintext, so the ambiguity the prefix leaves behind does not touch a single character that matters. Decryption runs the feedback in reverse: `pos(m_i) = pos(c_i) XOR s_i`, recover `m_i = key[pos(m_i)]`, then `s = pos(m_i)` and continue.

## Solve

```python
import itertools

CT = ('85677bc8302bb20f3be728f99be0002ee88bc8fdc045b80e1dd22bc8fcc0034dd809e8f77023fbc83'
      'cd02ec8fbb11cc02cdbb62837677bc8f2277eeaaaabb1188bc998087bef3bcf40683cd02eef48f44'
      'aaee805b8045453a546815639e6592c173e4994e044a9084ea4000049e1e7e9873fc90ab9e1d4437'
      'fc9836aa80423cc2198882a')
KNOWN = b'The secret message is:'.hex()   # 44 hex chars
HEX = '0123456789abcdef'
idx = {d: i for i, d in enumerate(HEX)}

# Equations: for each known i: pos(m_i) ^ pos(m_{i-1}) ^ pos(c_i) = 0 (i>=1),
# and pos(m_0) ^ pos(c_0) = 7. Duplicates toggle (XOR multiplicity).
eqs = []
def _mk(vars_):
    s = set()
    for v in vars_:
        s ^= {v}
    return s
eqs.append((_mk([idx[KNOWN[0]], idx[CT[0]]]), 7))
for i in range(1, len(KNOWN)):
    eqs.append((_mk([idx[KNOWN[i]], idx[KNOWN[i - 1]], idx[CT[i]]]), 0))

# Gaussian elimination over GF(2); each row is a 16-bit mask plus a 4-bit rhs.
rows = []
for vars_, rhs in eqs:
    mask = 0
    for v in vars_:
        mask |= 1 << v
    rows.append([mask, rhs])
pivot_of = {}
r = 0
for col in range(16):
    piv = next((j for j in range(r, len(rows)) if rows[j][0] >> col & 1), None)
    if piv is None:
        continue
    rows[r], rows[piv] = rows[piv], rows[r]
    for j in range(len(rows)):
        if j != r and (rows[j][0] >> col & 1):
            rows[j][0] ^= rows[r][0]
            rows[j][1] ^= rows[r][1]
    pivot_of[col] = r
    r += 1
free = [c for c in range(16) if c not in pivot_of]
print('free variables:', [HEX[c] for c in free])
assert all(m != 0 or rhs == 0 for m, rhs in rows), 'inconsistent system'

def decrypt(pos):
    key = [''] * 16
    for d, p in pos.items():
        key[p] = d
    s = 7
    out = []
    for c in CT:
        pm = pos[c] ^ s
        m = key[pm]
        out.append(m)
        s = pm
    return ''.join(out)

solutions = []
for free_vals in itertools.product(range(16), repeat=len(free)):
    posv = [0] * 16
    for c, v in zip(free, free_vals):
        posv[c] = v
    for col, ri in pivot_of.items():
        mask, rhs = rows[ri]
        v = rhs
        mm = mask & ~(1 << col)
        for c in free:
            if mm >> c & 1:
                v ^= posv[c]
        posv[col] = v
    if len(set(posv)) != 16:
        continue
    pos = {HEX[i]: posv[i] for i in range(16)}
    pt = decrypt(pos)
    if pt.startswith(KNOWN):
        solutions.append(pt)

uniq = sorted(set(solutions))
print('bijective completions:', len(solutions), 'unique plaintexts:', len(uniq))
for pt in uniq:
    secret_hex = pt[len(KNOWN):]
    print('plaintext hex:', pt)
    print('secret:', bytes.fromhex(secret_hex))
```

Run:

```text
$ python3 hexshift_solve.py
free variables: ['d', 'e', 'f']
bijective completions: 1344 unique plaintexts: 1
plaintext hex: 54686520736563726574206d6573736167652069733a204e696365206a6f6221204920686f706520796f7520656e6a6f79656420746865206368616c6c656e67652e2048657265277320796f757220666c61673a2044554354467b6431645f7930755f5573335f6775337373316e475f30725f6c316e3334725f346c67336252413f3f7d
secret: b' Nice job! I hope you enjoyed the challenge. Here\'s your flag: DUCTF{d1d_y0u_Us3_gu3ss1nG_0r_l1n34r_4lg3bRA??}'
```

## Flag

```text
DUCTF{d1d_y0u_Us3_gu3ss1nG_0r_l1n34r_4lg3bRA??}
```

Sanity check: 1,344 different bijective completions of the key all decrypt the handout to one identical plaintext, a grammatical message that ends by handing over the flag and then asking whether I used guessing or linear algebra. It was the algebra. The flag could not ride on a lucky key choice when every key the constraints allow tells the same story.
