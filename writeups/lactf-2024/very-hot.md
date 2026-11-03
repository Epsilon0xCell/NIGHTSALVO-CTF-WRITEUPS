# Very Hot

- **Event:** LA CTF 2024
- **Category:** crypto
- **Source:** official uclaacm/lactf-archive repo, `2024/crypto/very-hot`
- **Handouts:** `src.py`, `out.txt`

## Summary

RSA with a three prime modulus built from a single 384 bit prime: n = p * (p + 6) * (p + 12). Three factors huddled within 12 of each other means n is a hair away from a perfect cube, so the integer cube root of n lands within a few units of p. I scanned a small window around the cube root for a divisor, recovered all three primes from the offsets, and decrypted with the usual totient.

## First look

The generator spells out the structure:

```python
p = getPrime(384)
while(not isPrime(p + 6) or not isPrime(p + 12)):
    p = getPrime(384)
q = p + 6
r = p + 12

n = p * q * r
e = 2**16 + 1
ct = pow(FLAG, e, n)
```

The challenge title is the hint (these are "sexy primes", primes six apart, stacked three deep). The output file gives n at 1152 bits, e = 65537, and the ciphertext. Everything hinges on how close n sits to (p + 6)^3.

## What I tried first

The reflex for multi-prime RSA is to check whether the modulus leaks through a gcd or a small factor. n has no small factors, and there is no second ciphertext or related modulus to combine it with, so the standard RSA side doors are shut.

The other reflex is Fermat style factoring, hunting for n as a difference of squares. That works when two factors are close, but with three factors the geometry is cubic, not quadratic. The right generalization is to stop thinking of squares entirely and take the cube root: expanding (p + 6)^3 gives p^3 + 18p^2 + 216p + 216, while n expands to p^3 + 18p^2 + 72p. They differ by 144p - 216, which is nothing at 384 bit scale, so cbrt(n) should sit within a whisker of p + 6.

## The breakthrough

The integer cube root of n comes out 5 above p itself, so a scan of the integers within plus or minus 2000 of the cube root, testing divisibility, finds p almost immediately. I used a wide window because it costs nothing; the actual hit is at distance 5. From p, the other two factors are free: q = p + 6 and r = p + 12, and the product check p*q*r == n confirms the factorization rather than assuming it.

After that it is textbook RSA over three primes: phi = (p - 1)(q - 1)(r - 1), invert e, raise the ciphertext.

## Solve

```python
import re
from gmpy2 import iroot

txt = open('../handouts/very-hot-out.txt').read()
n = int(re.search(r'n: (\d+)', txt).group(1))
ct = int(re.search(r'ct: (\d+)', txt).group(1))
e = 65537

r, exact = iroot(n, 3)
print('integer cube root of n:', int(r), 'exact:', exact)
p = None
for cand in range(int(r) - 2000, int(r) + 2000):
    if cand > 0 and n % cand == 0:
        p = cand
        break
print('p =', p, ' (cube root overshoot:', int(r) - p, ')')
q, s = p + 6, p + 12
assert p * q * s == n
phi = (p - 1) * (q - 1) * (s - 1)
d = pow(e, -1, phi)
m = pow(ct, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, 'big').decode())
```

Output:

```
integer cube root of n: 21942765653871439764422303472543530148312720769660663866142363370143863717044484440248869144329425486818687730842082 exact: False
p = 21942765653871439764422303472543530148312720769660663866142363370143863717044484440248869144329425486818687730842077  (cube root overshoot: 5 )
lactf{th4t_w45_n0t_so_53xY}
```

## Flag

`lactf{th4t_w45_n0t_so_53xY}`
