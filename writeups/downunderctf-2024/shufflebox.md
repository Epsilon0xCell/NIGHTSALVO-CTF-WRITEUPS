# shufflebox

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** hashkitten

## Summary

"I've learned that if you shuffle your text, it's elrlay hrda to tlle htaw eht nioiglra nutpi aws." The handout shuffles text with a fixed character permutation and gives known plaintext/ciphertext pairs plus one censored line. Recover the text hidden behind the question marks and wrap it in `DUCTF{}`.

## First look

The publish folder only holds pointers to the source copies, so the real files are in `src/`:

```
$ cat shufflebox.py
import random

PERM = list(range(16))
random.shuffle(PERM)

def apply_perm(s):
	assert len(s) == 16
	return ''.join(s[PERM[p]] for p in range(16))

for line in open(0):
	line = line.strip()
	print(line, '->', apply_perm(line))

$ cat output_censored.txt
aaaabbbbccccdddd -> ccaccdabdbdbbada
abcdabcdabcdabcd -> bcaadbdcdbcdacab
???????????????? -> owuwspdgrtejiiud
```

Two things to note before touching anything. The permutation is shuffled fresh every time the generator runs (`random.shuffle` with no seed), so the `PERM` behind this handout is not in the code and cannot be regenerated. It has to be recovered from the two known pairs. And the direction matters: `apply_perm` builds the ciphertext as `cipher[p] = plain[PERM[p]]`, so cipher position `p` holds whichever plaintext character sits at index `PERM[p]`.

Reading the censored output literally gets nothing: `owuwspdgrtejiiud` is not text in any order I can eyeball.

## What I tried first

**One pair is not enough.** My first instinct was to read the permutation straight off the first pair. That fails on inspection: in `aaaabbbbccccdddd -> ccaccdabdbdbbada`, every plaintext character repeats four times, so pair 1 only tells me which letter group each cipher position draws from, not the exact index. The solve script counts it: 4 positions per group, four groups, so `4!^4 = 331,776` permutations stay consistent with pair 1 alone. A dead end, but a useful one: it says the second pair, with its `abcd` pattern repeated, is what separates indices inside each group.

**The wrong direction.** With a candidate `PERM` recovered, my first decode applied it to the censored text the same way the generator applies it to plaintext:

```
wrong-direction decode: teorjiwddpugsuiw
```

Still gibberish. That is the classic permutation trap: `PERM` answers "where does cipher position `p` read from", so decoding has to invert it, not replay it.

## The breakthrough

Combining both pairs pins every position: `PERM[p]` is the one plaintext index `i` such that `plain[i] == cipher[p]` in pair 1 **and** in pair 2. Running that match gives a unique index for all 16 positions, and re-encoding both known pairs with the recovered `PERM` reproduces the handout exactly, which is the check that the direction convention is right before I trust it on the censored line.

## Solve

```python
pairs = [
    ("aaaabbbbccccdddd", "ccaccdabdbdbbada"),
    ("abcdabcdabcdabcd", "bcaadbdcdbcdacab"),
]
censored = "owuwspdgrtejiiud"

# Step 1: how much does pair 1 alone pin down?
# The cipher position p holds plain[PERM[p]]. Pair 1 only tells us which
# letter group each cipher position draws from, not the exact index.
groups = {}
for p, ch in enumerate(pairs[0][1]):
    groups.setdefault(ch, []).append(p)
print("pair 1 alone, cipher positions grouped by source letter:", groups)
print("positions per group:", {k: len(v) for k, v in groups.items()},
      "-> 4!^4 =", 24**4, "permutations still fit pair 1")

# Step 2: PERM[p] is the plain index i such that for BOTH pairs,
# plain[i] == cipher[p]. With two pairs every position matches exactly one i.
PERM = []
for p in range(16):
    match = [i for i in range(16)
             if all(pt[i] == ct[p] for pt, ct in pairs)]
    assert len(match) == 1, (p, match)
    PERM.append(match[0])
print("recovered PERM (cipher position p takes plain[PERM[p]]):", PERM)

for pt, ct in pairs:
    assert ''.join(pt[PERM[p]] for p in range(16)) == ct
print("PERM reproduces both known pairs: OK")

# Step 3: WRONG direction first: apply PERM to the censored text directly,
# as if it mapped cipher positions back to plain positions.
wrong = ''.join(censored[PERM[p]] for p in range(16))
print("wrong-direction decode:", wrong)

# Step 4: correct inverse. plain[PERM[p]] = censored[p].
plain = ['?'] * 16
for p in range(16):
    plain[PERM[p]] = censored[p]
plain = ''.join(plain)
print("inverse decode:", plain)
print("flag: DUCTF{" + plain + "}")
```

Run:

```
$ python3 shufflebox_solve.py
pair 1 alone, cipher positions grouped by source letter: {'c': [0, 1, 3, 4], 'a': [2, 6, 13, 15], 'd': [5, 8, 10, 14], 'b': [7, 9, 11, 12]}
positions per group: {'c': 4, 'a': 4, 'd': 4, 'b': 4} -> 4!^4 = 331776 permutations still fit pair 1
recovered PERM (cipher position p takes plain[PERM[p]]): [9, 10, 0, 8, 11, 13, 3, 6, 15, 5, 14, 7, 4, 2, 12, 1]
PERM reproduces both known pairs: OK
wrong-direction decode: teorjiwddpugsuiw
inverse decode: udiditgjwowsuper
flag: DUCTF{udiditgjwowsuper}
```

## Flag

```
DUCTF{udiditgjwowsuper}
```

Sanity check: the recovered permutation re-encodes both known pairs to their exact published ciphertexts before it is ever applied to the censored line, and the decoded text reads as plain words in the handout's own style (`u did it gj wow super`).
