# Sun Zi's Perfect Math Class

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** wednesday

## Summary

A web lesson in two stages. Stage 1 is the classic Sun Zi / Han Xin remainder puzzle: find a number between 1000 and 1100 that leaves remainders 2, 4, 5 when divided by 3, 5, 7. Stage 2 explains the Chinese Remainder Theorem (CRT) and RSA, then gives one RSA message `m` encrypted with exponent `e = 3` under three different moduli `n1, n2, n3`, i.e. `c_i = m^3 mod n_i`. That is exactly the setup for Hastad's broadcast attack.

## Solve

Stage 1 is a direct search: the unique `x` in [1000, 1100] with `x % 3 == 2`, `x % 5 == 4`, `x % 7 == 5` is **1034** (check: 1034 = 3*344+2 = 5*206+4 = 7*147+5).

Stage 2: the stage data (`e = 3`, `c_1..c_3`, `n_1..n_3`, each `n_i` 1024 bits) is hardcoded in the page source, and the challenge's own generator confirms the construction (e = 3, three moduli). With `e` encryptions of the same message under `e` pairwise coprime moduli, CRT recombines the ciphertexts into `m^e` modulo `N = n1*n2*n3`, and because `m^3 < N`, that value is `m^3` over the integers: a plain integer cube root recovers `m`.

```python
import re

# c_i, n_i parsed from the page source with a regex (no hand transcription)
def crt(c, n):
    N = n[0] * n[1] * n[2]
    x = 0
    for ci, ni in zip(c, n):
        Mi = N // ni
        x += ci * Mi * pow(Mi, -1, ni)   # inverse via extended Euclid
    return x % N

x = crt([c1, c2, c3], [n1, n2, n3])      # x = m^3, here 1267 bits < N

def icbrt(a):                            # integer cube root, Newton iteration
    x = 1 << ((a.bit_length() + 2) // 3)
    while True:
        y = (2 * x + a // (x * x)) // 3
        if y >= x:
            return x
        x = y

m = icbrt(x)
assert m ** 3 == x                       # exact, verified
flag = m.to_bytes((m.bit_length() + 7) // 8, 'big')
```

Observed output:

```
stage1 soldiers: 1034
CRT combined bits 1267
m^3 == CRT ? True
decoded: b'DUCTF{btw_y0u_c4n_als0_us3_CRT_f0r_p4rt14l_fr4ct10ns}'
```

## Flag

```
DUCTF{btw_y0u_c4n_als0_us3_CRT_f0r_p4rt14l_fr4ct10ns}
```
