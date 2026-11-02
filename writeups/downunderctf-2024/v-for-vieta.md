# V for Vieta

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** easy
- **Author:** wednesday

## Summary

An interactive server deals rounds of a number theory challenge. Each round it picks a secret `r`, sets `k = r^2`, and demands two integers `a, b` (each over 2048 bits) with

```
(a^2 + a*b + b^2) / (2*a*b + 1) = k
```

The bit-length bar rises as rounds progress, and the flag comes after surviving every round.

## Solve

Fix `k`. Solutions of `a^2 + a*b + b^2 = k*(2*a*b + 1)` come in Vieta-jumping chains: treat the equation as a quadratic in `a` for fixed `b`. If `(a, b)` is a solution, the other root is

```
a' = 2*k*b - b - a
```

which is again a non-negative integer solution. There is a seed solution on the ground floor: `(0, sqrt(k))`, and `sqrt(k) = r` exactly because `k = r^2` (check: `(0 + 0 + r^2) / (0 + 1) = r^2 = k`). Alternating the jump on each coordinate climbs the chain, and the values grow geometrically, so a few dozen jumps take both `a` and `b` past 2048 bits.

```python
from math import isqrt

def climb(k, target_bits=2049):
    a, b = 0, isqrt(k)
    assert a*a + a*b + b*b == k * (2*a*b + 1)
    while a.bit_length() < target_bits or b.bit_length() < target_bits:
        a = 2*k*b - b - a      # Vieta jump on a
        a, b = b, a            # then jump the other coordinate next round
        assert a*a + a*b + b*b == k * (2*a*b + 1)
    return a, b
```

The solver script played the server locally: read `k` each round, climb from the seed, assert the equation holds locally before answering, and send `(a, b)`. 27 messages end to end, then the server printed the flag.

## Flag

```
DUCTF{jump1n6_4nd_fl1pp1n6_up_7h3_hyp3r8011c_14dd3r}
```
