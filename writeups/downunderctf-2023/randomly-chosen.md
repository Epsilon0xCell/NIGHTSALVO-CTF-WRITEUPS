# Randomly Chosen

- **Event:** DownUnderCTF 2023
- **Category:** beginner (crypto)
- **Source:** official DownUnderCTF/Challenges_2023_Public repo, `beginner/randomly-chosen`
- **Handouts:** `randomly-chosen.py`, `output.txt`

## Summary

The generator seeds Python's `random` module with a value drawn from a range of only 1337 possibilities, then publishes 305 characters sampled with replacement from the flag. I replayed every possible inner seed, rebuilt the exact index sequence each one produces, and kept the seed whose sequence maps the output onto a single consistent flag. Exactly one seed survives, and it hands back the whole 61 character flag.

## First look

The generator, in full:

```python
import random

random.seed(random.randrange(0, 1337))
flag = open('./flag.txt', 'r').read().strip()
out = ''.join(random.choices(flag, k=len(flag)*5))
print(out)
```

Two facts fall out of the source. The output is 305 characters, and since the sample count is `len(flag) * 5`, the flag is 61 characters long. And `random.choices` picks characters from the flag string itself, so every output character is a flag character, positioned by an index sequence that depends only on the seed and the flag length.

## What I tried first

My first framing of the problem was wrong in a subtle way. I wanted to reproduce the server's exact seeding path: seed the generator, call `randrange(0, 1337)`, seed again. That path is unreproducible on purpose, because the outer seed came from operating system entropy at generation time. Chasing it is chasing a number nobody recorded.

The useful reframing is that the outer call only chooses which of 1337 inner seeds was used. Whatever entropy fed the first seeding, the second seed is an integer in [0, 1337), and from that integer onward everything is deterministic. So the search space is 1337 seeds, not the whole entropy pool.

A second false start was statistical: the output is a sample of the flag's characters, so letter frequencies leak the flag's character multiset. That gives a bag of characters with no order. Interesting, but 61 characters is too much ordering information to recover from a bag, and the seed attack makes it unnecessary.

## The breakthrough

For a candidate seed s and length L = 61, I can generate the exact index sequence the server used by calling `random.choices(range(L), k=305)` after `random.seed(s)`: `choices` selects indices by the same draws whether the population is a string or a range. Each output character then constrains one flag position: out[j] must equal flag[idx[j]]. If two draws land on the same index with different output characters, the seed is impossible. Most seeds die within the first few dozen draws.

Adding the known frame (DUCTF{ at the front, } at the end) prunes further, but the satisfying result is that it is barely needed: with the frame check removed entirely, exactly one seed, 252, produces a consistent mapping at all. The output alone pins the seed; the frame just confirms it.

## Solve

```python
import random

out = open('../handouts/randomly-chosen-output.txt').read().strip()
print('output length:', len(out))
L = len(out) // 5                      # out has len(flag) * 5 characters
assert len(out) == 5 * L
known = {0: 'D', 1: 'U', 2: 'C', 3: 'T', 4: 'F', 5: '{', L - 1: '}'}
for seed in range(1337):               # the inner seed is randrange(0, 1337)
    random.seed(seed)
    idx = random.choices(range(L), k=len(out))
    flag = [None] * L
    ok = True
    for j, i in enumerate(idx):
        if flag[i] is None:
            flag[i] = out[j]
        elif flag[i] != out[j]:
            ok = False
            break
    if not ok:
        continue
    if any(flag[p] is not None and flag[p] != c for p, c in known.items()):
        continue
    print('seed', seed, '->', ''.join(flag))
```

Output:

```
output length: 305
seed 252 -> DUCTF{is_r4nd0mn3ss_d3t3rm1n1st1c?_cba67ea78f19bcaefd9068f1a}
```

Every flag position gets sampled several times over 305 draws, so the reconstruction is total rather than partial: there are no gaps to guess.

## Flag

`DUCTF{is_r4nd0mn3ss_d3t3rm1n1st1c?_cba67ea78f19bcaefd9068f1a}`
