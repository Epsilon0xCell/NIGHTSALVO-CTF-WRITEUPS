# baby-arx

- **Event:** DownUnderCTF 2022
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** joseph#8210

## Summary

"ARX is just add-rotate-xor right? But what if it's just rotate-xor?" The handout is the cipher source, a 64-byte keystream printed as a hex comment, and the note "to get the flag just print the state." The state is initialized to the flag itself and then scrambled through two byte functions. Both functions turn out to be linear over GF(2), and linear maps over bits can be walked backwards.

## First look

```python
def g(x):
    return x ^ ((x << 1) & 0xFF) ^ (x & 1)

def h(y):
    return y ^ ((y << 3) & 0xFF) ^ (y >> 5)

state = list(flag)
out = []
for i in range(len(state) - 1):
    out.append((g(state[i]) + h(state[i + 1])) % 256)
out.append((g(state[-1]) + h(out[0])) % 256)
## e.g. keystream = 8ae32e1b...
```

Parsing the loop: output byte `i` (for `i` up to 62) is `g(F[i]) + h(F[i+1]) mod 256`, a sum of one function of the current flag byte and one function of the next. The last output byte wraps around: `out[63] = g(F[63]) + h(out[0])`, which ties the end of the chain back to the start and will make a closing check. The known prefix `DUCTF{` gives me the first six bytes of the state for free.

## What I tried first

**Per-byte inversion.** The natural first move is to invert `g` and `h` separately and peel the chain. It fails on a structural fact: both maps are linear over GF(2) with a nontrivial kernel. Working out the bit matrices shows `h(y) = y XOR ROL(y, 3)`, whose kernel is `{0x00, 0xFF}` (any input with all bits equal maps to zero), and `g` has the same kernel shape. So every output byte value has exactly two preimages under each function, `x` and `x XOR 0xFF`, and "the" inverse does not exist. A byte-by-byte peel branches or dies immediately.

**Full linear algebra over the whole state.** Treating all 512 state bits as unknowns and solving the output equations as one GF(2) system is clean in theory. In practice the bytewise addition mod 256 sits between the two halves of every equation, and addition carries are not GF(2)-linear, so the system does not assemble. The chain structure is the way in.

## The breakthrough

Use the sum, not the parts. From `out[i] = g(F[i]) + h(F[i+1])`, knowing `F[i]` leaves `h(F[i+1]) = out[i] - g(F[i])`, and since `h` has exactly two preimages per value, `F[i+1]` has exactly two candidates. Walk the chain from the known `D` at position 0: at every step, branch over at most two candidates and keep only the printable ones (flag bytes are printable ASCII). The branching stays tiny (the run reports two states alive after the first step, one or two at each step after), and two closing conditions select the unique survivor: the wrap equation `out[63] = g(F[63]) + h(out[0])` must hold, and the recovered state must end in `}`. One candidate passes everything; the uniqueness is the proof.

## Solve

```python
"""Solver: walk the ARX chain forward from the known prefix, branching over
the two preimages of h at each step and pruning to printable bytes."""
KEYSTREAM = bytes.fromhex(
    '8ae32e1b16923bf07f1a82b4612ff3b2fdd4648b58436cb43eda2b54891ce992'
    '0a915a985b69c61d5f34ef2097aebfc3f58769958d46a1c78003618640db99da')

def g(x):
    return (x ^ ((x << 1) & 0xFF) ^ (x & 1)) & 0xFF

def rol8(x, r):
    return ((x << r) | (x >> (8 - r))) & 0xFF

def h(y):
    return (y ^ rol8(y, 3)) & 0xFF

H_PRE = {}
for y in range(256):
    H_PRE.setdefault(h(y), []).append(y)

assert set(len(v) for v in H_PRE.values()) == {2}  # kernel {0, 0xff}
n = len(KEYSTREAM)
states = [(ord('D'),)]  # state[0] known from the DUCTF{ prefix
for i in range(n - 1):
    nxt = []
    for st in states:
        need = (KEYSTREAM[i] - g(st[i])) % 256
        for cand in H_PRE[need]:
            if 32 <= cand < 127:
                nxt.append(st + (cand,))
    states = nxt
    print(f'after byte {i + 1}: {len(states)} candidate state(s)')
    assert states, 'chain died'

sols = []
for st in states:
    if (g(st[-1]) + h(KEYSTREAM[0])) % 256 != KEYSTREAM[-1]:
        continue
    if st[-1] != ord('}'):
        continue
    sols.append(bytes(st))
print('solutions:', len(sols))
for s in sols:
    print('flag:', s)
```

Run:

```text
$ python3 babyarx_solve.py
after byte 1: 2 candidate state(s)
...
after byte 63: 2 candidate state(s)
solutions: 1
flag: b'DUCTF{i_d0nt_th1nk_th4ts_h0w_1t_w0rks_actu4lly_92f45fb961ecf420}'
```

(The middle lines are one "after byte i: 1 or 2 candidate state(s)" line per step; the count never exceeds 2 on the whole walk.)

## Flag

```text
DUCTF{i_d0nt_th1nk_th4ts_h0w_1t_w0rks_actu4lly_92f45fb961ecf420}
```

Sanity check: the solution is unique, and it is validated by conditions the walk never used as input: the wrap-around equation involving the first keystream byte must hold for the final byte, and the state must close with `}` and start with the full `DUCTF{` prefix. A wrong branch anywhere in the middle breaks the printable filter or the closing equation, so a single survivor that passes everything is the state the challenge generated.
