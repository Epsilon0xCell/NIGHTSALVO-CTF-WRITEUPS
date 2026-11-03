# Complementary

- **Event:** DownUnderCTF 2023
- **Category:** beginner (crypto)
- **Source:** official DownUnderCTF/Challenges_2023_Public repo, `beginner/complementary`
- **Handouts:** `complementary.py`, `output.txt`

## Summary

The script splits the flag in half, reads each half as a big integer, and prints their product. That makes the whole challenge one factoring problem, and the punchline is that the product is not a hard semiprime at all: the two halves, being ASCII, make a number that standard factoring tools split in half a second. After that, a subset search over the prime factors finds the split where one side starts with DUCTF{ and the other ends with a closing brace.

## First look

The entire generator is four lines:

```python
flag = open('./flag.txt', 'rb').read().strip()
m1 = int.from_bytes(flag[:len(flag)//2])
m2 = int.from_bytes(flag[len(flag)//2:])
n = m1 * m2
print(n)
```

The output is one 88 digit number:

```
6954494065942554678316751997792528753841173212407363342283423753536991947310058248515278
```

## What I tried first

My first read was pessimistic. If m1 and m2 were random numbers of this size, n would be a roughly 292 bit semiprime with two balanced factors near 146 bits each, and factoring that with desktop tools is not a beginner task. I spent the first minutes considering the known-plaintext angle instead: m1 begins with the bytes DUCTF{, which fixes the top 48 bits of one factor and points at a Coppersmith style lattice attack.

Before building any lattice, I did the cheap thing and let a general factoring routine look at n. That should have been step one from the start. Trial division finds a factor of 2 immediately, the number is even, and a balanced semiprime of two odd primes cannot be even. The pessimistic read was wrong in the first ten seconds of testing.

## The breakthrough

The number is highly composite. Running sympy's `factorint` (trial division, Pollard rho, and ECM in one pipeline) returns the full factorization in 0.5 seconds:

```
{2: 1, 3: 1, 19: 1, 31: 1, 83: 1, 3331: 1, 165219437: 1, 550618493: 1,
 66969810339969829: 1, 1168302403781268101731523384107546514884411261: 1}
```

That is the real lesson of the challenge. m1 and m2 are not random; they are text. Two ASCII strings read as integers carry small factors at a rate random numbers never would, so the product inherits a rich factorization even though neither half is itself factorable by inspection. Ten distinct prime factors means 2^10 ways to split them into two groups, and only one grouping decodes as the two halves of a flag: one side's bytes start with DUCTF{, the other side's bytes end with }, and everything is printable.

## Solve

```python
from sympy import factorint
from itertools import combinations

n = int(open('../handouts/complementary-output.txt').read().strip())
f = factorint(n)
print('factorization:', f)
prod = 1
for k, e in f.items():
    prod *= k ** e
assert prod == n, 'product check failed'

primes = []
for k, e in f.items():
    primes += [k] * e
print('prime factors with multiplicity:', len(primes))

def as_bytes(x):
    return x.to_bytes((x.bit_length() + 7) // 8, 'big')

answer = None
for r in range(1, len(primes)):
    for idxs in combinations(range(len(primes)), r):
        m1 = 1
        for i in idxs:
            m1 *= primes[i]
        b1 = as_bytes(m1)
        if not b1.startswith(b'DUCTF{'):
            continue
        b2 = as_bytes(n // m1)
        if b2.endswith(b'}') and all(32 <= ch < 127 for ch in b1 + b2):
            answer = b1 + b2
            break
    if answer:
        break
print(answer.decode())
```

Output:

```
factorization: {2: 1, 3: 1, 19: 1, 31: 1, 83: 1, 3331: 1, 165219437: 1, 550618493: 1, 66969810339969829: 1, 1168302403781268101731523384107546514884411261: 1}
prime factors with multiplicity: 10
DUCTF{is_1nt3ger_f4ct0r1s4t10n_h4rd?}
```

## Flag

`DUCTF{is_1nt3ger_f4ct0r1s4t10n_h4rd?}`

Apparently not.
