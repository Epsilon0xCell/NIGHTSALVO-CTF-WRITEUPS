# apbq RSA I

- **Event:** DownUnderCTF 2023
- **Category:** crypto
- **Source:** official DownUnderCTF/Challenges_2023_Public repo, `crypto/apbq-rsa-i`
- **Handouts:** `apbq-rsa-i.py`, `output.txt`

## Summary

A 2048 bit RSA modulus with two hints of the form a*p + b*q, where each a is at most 12 bits and each b at most 312 bits. Eliminating the b terms between the two hints leaves a multiple of q whose cofactor is small enough to search: for the right pair (a1, a2), gcd(a1*h2 - a2*h1, n) is q itself. I found the pair with a batch gcd search over the 4097 x 4097 grid, factored n, and decrypted. The winning pair is a1 = 410, a2 = 761.

## First look

The generator:

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 0x10001

hints = []
for _ in range(2):
    a, b = randint(0, 2**12), randint(0, 2**312)
    hints.append(a * p + b * q)
```

So h1 = a1*p + b1*q and h2 = a2*p + b2*q, with the a values tiny and the b values merely moderate. The asymmetry is the whole challenge: 12 bits is a searchable space, 312 bits is not.

## What I tried first

The free checks first. If either hint had a1 or a2 equal to zero it would be a pure multiple of one prime, so I computed gcd(h1, n) and gcd(h2, n). Both are 1. No luck there.

Next I measured the honest brute force: loop over every (a1, a2) pair, compute the combination, take a gcd with n. Sampling 2000 gcd checks against the 2048 bit modulus took 0.03 seconds under gmpy2, which projects the full 16.8 million pair grid at roughly 4 minutes. Feasible, but wasteful, because almost every pair produces a combination coprime to n and the gcd work is thrown away. That measurement is what pushed me to batch the search instead of waiting it out.

## The breakthrough

Multiply the first hint by a2 and the second by a1 and subtract:

a1*h2 - a2*h1 = (a1*b2 - a2*b1) * q

The p terms cancel exactly. For the true pair, the combination is a multiple of q, so its gcd with n = p*q is q (the cofactor a1*b2 - a2*b1 is far too small to also contain p). For wrong pairs the combination is a random looking number, coprime to n with overwhelming probability.

That turns the grid search into a filtering problem. Fix a1. The values v(j) = a1*h2 - j*h1 for j in [0, 4097) are almost all coprime to n, so the gcd of their product (taken mod n) with n is 1 unless the true a2 is in the range. Recursively halving the range and re-testing products pinpoints a2 in a few dozen gcds instead of thousands. The first a1 with a hit is the answer.

## Solve

```python
import re, sys
from gmpy2 import gcd, mpz

txt = open('../handouts/apbq-rsa-i-output.txt').read()
n = mpz(re.search(r'n = (\d+)', txt).group(1))
c = mpz(re.search(r'c = (\d+)', txt).group(1))
h1, h2 = [mpz(x) for x in re.search(r'hints = \[(\d+), (\d+)\]', txt).groups()]
e = 0x10001

print('gcd(h1, n) =', gcd(h1, n))
print('gcd(h2, n) =', gcd(h2, n))

sys.setrecursionlimit(10000)

def find(a1, lo, hi):
    if hi - lo == 1:
        g = gcd(a1 * h2 - lo * h1, n)
        return lo if 1 < g < n else None
    mid = (lo + hi) // 2
    for a, b in ((lo, mid), (mid, hi)):
        P = mpz(1)
        for j in range(a, b):
            P = P * ((a1 * h2 - j * h1) % n) % n
        if gcd(P, n) > 1:
            r = find(a1, a, b)
            if r is not None:
                return r
    return None

found = None
for a1 in range(4097):
    a2 = find(a1, 0, 4097)
    if a2 is not None:
        q = gcd(a1 * h2 - a2 * h1, n)
        p = n // q
        assert p * q == n
        print(f'a1 = {a1}, a2 = {a2}')
        found = (p, q)
        break
p, q = found
d = pow(e, -1, (p - 1) * (q - 1))
m = pow(c, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, 'big').decode())
```

Output:

```
gcd(h1, n) = 1
gcd(h2, n) = 1
a1 = 410, a2 = 761
DUCTF{gcd_1s_a_g00d_alg0r1thm_f0r_th3_t00lbox}
```

The whole search finishes in a few seconds. The flag is right about the tool that did it.

## Flag

`DUCTF{gcd_1s_a_g00d_alg0r1thm_f0r_th3_t00lbox}`
