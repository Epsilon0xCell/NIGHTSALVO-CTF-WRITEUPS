# Sun Zi's Perfect Math Class

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** wednesday

## Summary

A web lesson in two stages. Stage 1 is the classic Sun Zi / Han Xin remainder puzzle: find a number between 1000 and 1100 that leaves remainders 2, 4, 5 when divided by 3, 5, 7. Stage 2 explains the Chinese Remainder Theorem (CRT) and RSA, then gives one RSA message `m` encrypted with exponent `e = 3` under three different moduli `n1, n2, n3`, i.e. `c_i = m^3 mod n_i`. That is exactly the setup for Hastad's broadcast attack.

## First look

There is no publish handout for this one; the challenge is a web page built from `src/`. The interesting files:

```
$ find publish src -type f
src/src/App.tsx
src/utils/generate_hastad.py
src/package.json
```

`src/utils/generate_hastad.py` is the author's own generator, and it confirms the construction with no guessing: it takes the flag as `m`, sets `e = 3`, builds three moduli as products of two 512-bit primes, encrypts the same `m` under each, then CRT-combines and takes an integer cube root to check itself. The live numbers are hardcoded in `src/src/App.tsx` inside `Stage2Directions` (lines 208 onwards give `c_1 = 105001824161664003599422656864176455171381720653815905925856548632486703162518989165039084097502312226864233302621924809266126953771761669365659646250634187967109683742983039295269237675751525196938138071285014551966913785883051544245059293702943821571213612968127810604163575545004589035344590577094378024637`, and the rest the same way), and the App code pins stage 1 at `mods = [3, 5, 7]`, `rems = [2, 4, 5]`. I parsed the numbers out of `App.tsx` with a regex rather than transcribing 300-digit constants by hand.

## What I tried first

**Stage 1 by hand, briefly.** The lesson wants you to feel the CRT idea, but with a range of 101 candidates the honest first move is a brute force loop. It returns exactly one survivor, 1034. Fine, stage 1 done, and it warms up the real point: remainders under coprime moduli pin a number down.

**A shared-factor shortcut on stage 2?** Before doing the attack properly I checked whether the three moduli share a prime (a depressingly common RSA breakage, and CRT needs coprime moduli anyway):

```
gcd(n1,n2) = 1  gcd(n1,n3) = 1  gcd(n2,n3) = 1 -> pairwise coprime, no shared prime to split
```

No luck there, which is also the green light for CRT.

**Cube root of one ciphertext.** The next lazy idea is to take an integer or float cube root of `c_1` alone. Both fail, for the record:

```
float cube root of c1: 4.71772130027414e+102
int(that)^3 == c1 ? False
icbrt(c1)^3 == c1 ? False -> a single ciphertext is not a perfect cube
```

The float root is only a 53-bit-mantissa sketch of a 1024-bit number, and the exact integer root confirms `c_1` is not a perfect cube at all, because `c_1 = m^3 mod n_1`, not `m^3`. One ciphertext by itself is a dead end by construction.

## The breakthrough

Three ciphertexts of the **same** message under three coprime moduli is the whole attack. CRT recombines them into one value `X = m^3 mod (n1*n2*n3)`. The flag is short (a few dozen bytes), so `m^3` is only about 1267 bits, far below the 3070-bit product `N`. That means the modular reduction never wraps: `X` is not `m^3 mod N`, it is `m^3` itself, over the integers, and a plain integer cube root hands back `m`. The script's printed bit lengths are the moment this lands: CRT combined value 1267 bits against `N` at 3070 bits.

## Solve

```python
import re
from math import gcd

text = open('src/src/App.tsx').read()

def get(name):
    return int(re.search(name + r"\s*=\s*(\d+)", text).group(1))

e = 3
cs = [get("c_1"), get("c_2"), get("c_3")]
ns = [get("n_1"), get("n_2"), get("n_3")]
print("parsed from App.tsx: e =", e)
for i, (c, n) in enumerate(zip(cs, ns), 1):
    print(f"  c{i}: {c.bit_length()} bits, n{i}: {n.bit_length()} bits")

# Stage 1: the Han Xin soldier count, brute force over the stated range.
sol = [x for x in range(1000, 1101)
       if x % 3 == 2 and x % 5 == 4 and x % 7 == 5]
print("stage1 soldiers:", sol)
x = sol[0]
print(f"  check: {x} = 3*{x//3}+{x%3} = 5*{x//5}+{x%5} = 7*{x//7}+{x%7}")

# Dead end 1: shared-factor shortcut? CRT needs coprime moduli anyway.
print("gcd(n1,n2) =", gcd(ns[0], ns[1]),
      " gcd(n1,n3) =", gcd(ns[0], ns[2]),
      " gcd(n2,n3) =", gcd(ns[1], ns[2]),
      "-> pairwise coprime, no shared prime to split")

# Dead end 2: naive float cube root of one ciphertext. Floats carry 53
# bits of mantissa, so the root is a blurry approximation and rounding
# it does not cube back to c1.
r = cs[0] ** (1 / 3)
print("float cube root of c1:", r)
print("int(that)^3 == c1 ?", int(r) ** 3 == cs[0])

# Dead end 3: integer cube root of c1 alone is not exact, because
# c1 = m^3 mod n1, not m^3 itself.
def icbrt(a):
    x = 1 << ((a.bit_length() + 2) // 3 + 1)
    while True:
        y = (2 * x + a // (x * x)) // 3
        if y >= x:
            break
        x = y
    while x ** 3 > a:
        x -= 1
    while (x + 1) ** 3 <= a:
        x += 1
    return x

r1 = icbrt(cs[0])
print("icbrt(c1)^3 == c1 ?", r1 ** 3 == cs[0],
      "-> a single ciphertext is not a perfect cube")

# The attack: CRT recombines the three ciphertexts into m^3 mod N,
# and m^3 < N, so the result is m^3 over the integers.
N = ns[0] * ns[1] * ns[2]
X = 0
for ci, ni in zip(cs, ns):
    Mi = N // ni
    X += ci * Mi * pow(Mi, -1, ni)
X %= N
print("CRT combined value: ", X.bit_length(), "bits, N is", N.bit_length(), "bits")

m = icbrt(X)
print("m^3 == CRT value ?", m ** 3 == X)
print("check pow(m,3,n_i) == c_i for all i:",
      all(pow(m, 3, ni) == ci for ci, ni in zip(cs, ns)))
flag = m.to_bytes((m.bit_length() + 7) // 8, 'big')
print("decoded:", flag)
```

Run:

```
$ python3 sunzi_solve.py
parsed from App.tsx: e = 3
  c1: 1024 bits, n1: 1024 bits
  c2: 1022 bits, n2: 1024 bits
  c3: 1023 bits, n3: 1024 bits
stage1 soldiers: [1034]
  check: 1034 = 3*344+2 = 5*206+4 = 7*147+5
gcd(n1,n2) = 1  gcd(n1,n3) = 1  gcd(n2,n3) = 1 -> pairwise coprime, no shared prime to split
float cube root of c1: 4.71772130027414e+102
int(that)^3 == c1 ? False
icbrt(c1)^3 == c1 ? False -> a single ciphertext is not a perfect cube
CRT combined value:  1267 bits, N is 3070 bits
m^3 == CRT value ? True
check pow(m,3,n_i) == c_i for all i: True
decoded: b'DUCTF{btw_y0u_c4n_als0_us3_CRT_f0r_p4rt14l_fr4ct10ns}'
```

## Flag

```
DUCTF{btw_y0u_c4n_als0_us3_CRT_f0r_p4rt14l_fr4ct10ns}
```

Sanity check: the recovered `m` is verified in both directions, `m^3` equals the CRT combination exactly, and re-encrypting `m` with `pow(m, 3, n_i)` reproduces all three published ciphertexts bit for bit.
