# Substitution Cipher I

- **Event:** DownUnderCTF 2021
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** joseph
- **Source:** official DownUnderCTF/Challenges_2021_Public repo, `crypto/substitution-cipher-i`
- **Handouts:** `substitution-cipher-i.sage`, `output.txt`

## Summary

The handout script pushes every character of the flag through one polynomial, f(x) = 13x^2 + 3x + 7, and prints the results as Unicode characters. Because f is strictly increasing over the byte range, it is a substitution cipher with a formula instead of a table, and I inverted it character by character by solving the quadratic.

## First look

The Sage source is five lines of substance:

```python
def encrypt(msg, f):
    return ''.join(chr(f.substitute(c)) for c in msg)

P.<x> = PolynomialRing(ZZ)
f = 13*x^2 + 3*x + 7

FLAG = open('./flag.txt', 'rb').read().strip()
enc = encrypt(FLAG, f)
print(enc)
```

The output file holds 28 characters with code points like 60323, 94187, 58565. Those numbers are far outside ASCII, which fits: f(68) for the letter D is already 13*68^2 + 3*68 + 7 = 60323. The first output code point matches f('D') exactly, a good sign before writing any solver.

## What I tried first

My first instinct was to treat it as an affine map and fit a line through the first two points, assuming they were D and U from the flag format. That gives a slope of (94187 - 60323) / (85 - 68) = 1992 and an intercept of -75133. The fit dies on the third character: it predicts 1992*67 - 75133 = 58331 for C, but the actual third code point is 58565. Not affine, which the quadratic in the source already said, but I wanted the numbers to confirm it before trusting the source reading.

Frequency analysis was the other reflex, and it is pointless here: 28 characters is too short, and the mapping is deterministic arithmetic rather than a shuffled alphabet, so there is nothing statistical to squeeze.

## The breakthrough

For positive x, f'(x) = 26x + 3 is positive, so f is one-to-one on byte values and each output has exactly one preimage. Solving 13x^2 + 3x + 7 = y for x gives a discriminant of 52y - 355, which must be a perfect square, and x = (sqrt(52y - 355) - 3) / 26. The first character checks out by hand: 52*60323 - 355 = 3136441 = 1771^2, and (1771 - 3)/26 = 68, the letter D.

## Solve

```python
import math

data = open('../handouts/substitution-cipher-i-output.txt', encoding='utf-8').read().strip()
out = []
for ch in data:
    y = ord(ch)
    disc = 52 * y - 355            # 13x^2 + 3x + 7 = y  =>  disc = 9 - 52*(7 - y)
    s = math.isqrt(disc)
    assert s * s == disc, (y, disc)
    x, rem = divmod(s - 3, 26)
    assert rem == 0
    out.append(x)
print(bytes(out).decode())
```

The assertions do the verification work: every discriminant is a perfect square and every division is exact, so the decode is not a best guess, it is forced. Output:

```
DUCTF{sh0uld'v3_us3d_r0t_13}
```

The flag argues I should have used ROT13. A quadratic with a closed-form inverse is not much of an upgrade.

## Flag

`DUCTF{sh0uld'v3_us3d_r0t_13}`
