# DNAdecay

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** easy
- **Author:** BootlegSorcery@

## Summary

"Our flightless birds can run upto 50km/h but we want them to go faster. I've been messing with a mutigen but it seems to have corrupted. Can you help me recover this research?" The handout is a single file, `dna.rb`: 236 lines of ASCII art shaped like a DNA double helix, with some of the base letters rotted away to spaces. Recover the program the art encodes.

## First look

```text
$ head -8 publish/dna.rb
require "doublehelix"
   A--T
  G---C

 A----T
  T----A
   C---G
    G--C
```

The `require` line gives the game away. The Ruby `doublehelix` gem (by Yusuke Endoh, mame/doublehelix) obfuscates a Ruby program as DNA art: it takes the source bytes as an LSB-first bit stream (`src.unpack("b*")`), maps each pair of bits to a base pair (00 to AT, 01 to CG, 10 to GC, 11 to TA), and draws one rung per pair. The rung shapes rotate through an 18-entry format table of (offset, dash count) pairs: the first nine entries are `[1,0], [0,2], [0,3], [0,4], [1,4], [2,4], [3,3], [4,2], [5,0]` and the next nine are the same list reversed. So for rung `i`, the two bases sit at exact columns: `offset` and `offset + 1 + dist`. That is the whole encoding, and it means decoding is a parsing problem, not an image problem.

## What I tried first

**Treating surviving bases as the data.** My first decoder read each rung's two visible character slots and mapped whatever letter it found straight into the pair table. Where a base had decayed to a space I guessed which side of the rung it belonged on. The output was garbage in patches: the orientation model was wrong, because a rung like `G--C` and a rung like `C--G` are different pairs, and a lone surviving base does not say which end of the rung it came from unless you use its exact column. Bases sit at rung-specific columns, and a letter in the left column means something different from the same letter in the right column. Guessing sides corrupts about half the singly-decayed rungs.

**Assuming the decay was fatal.** Counting spaces, a fair number of rungs have lost a base and eight rungs have lost both. It is tempting to call those bits gone. They are not, mostly: a base pair is complementary (A pairs with T, C pairs with G), so either surviving base determines the entire pair and both of its bits. Only the both-bases-gone rungs lose information, and there are just eight of those in 236 rungs.

## The breakthrough

Parse by exact columns, using the gem's own format table, and let complementarity do the repair. For each rung, read the characters at the two computed columns and nothing else. If either is a real base, the pair is fixed; if both are spaces, mark two bits unknown and move on. Doing that across all 236 rungs gives zero column misalignments and a bit stream that is complete except for 16 unknown bits sitting in 8 rungs, touching 7 bytes of the 59-byte program. The skeleton already reads as a sentence with holes:

```text
skeleton: puts"DUCTF{7H3_Mit0.HOn.Ri4.15.7he_P0wEr_HoU.E_of.DA_C3L.}"
```

Each holed byte has only a handful of candidates consistent with its surviving bits (byte 56 is forced outright), and the text is leetspeak, so each hole has exactly one reading that makes a word: Mit0cHOndRi4, 15_7he, HoUsE, of_DA, C3LL. The candidates and picks:

- byte 19 in {c, g, k, o} goes to c ("Mit0cHOndRi4", mitochondria)
- byte 23 in {d, e, f, g} goes to d
- byte 27 in {/, ?, O, _, o} goes to _ (word break)
- byte 30 in {O, _, o} goes to _ ("15_7he")
- byte 44 in {p, q, r, s} goes to s ("HoUsE")
- byte 49 in {O, _, o} goes to _ ("of_DA")
- byte 56 in {L} is forced

## Solve

```python
import re

fmt9 = [[1, 0], [0, 2], [0, 3], [0, 4], [1, 4], [2, 4], [3, 3], [4, 2], [5, 0]]
FMT = fmt9 + fmt9[::-1]

PAIR_FROM_FIRST = {'A': 'AT', 'T': 'TA', 'C': 'CG', 'G': 'GC'}
PAIR_FROM_SECOND = {'T': 'AT', 'A': 'TA', 'G': 'CG', 'C': 'GC'}
BITS = {'AT': '00', 'CG': '01', 'GC': '10', 'TA': '11'}       # gem decoder dict
PAIR_BITS = {'00': 'AT', '01': 'CG', '10': 'GC', '11': 'TA'}  # gem encoder dict

lines = open('publish/dna.rb').read().split('\n')
assert lines[0].startswith('require')
art = lines[1:]
while art and art[0] == '':
    art.pop(0)
while art and art[-1] == '':
    art.pop()
print('art lines (rungs):', len(art))

pairs, unknown_rungs = [], []
for i, ln in enumerate(art):
    off, dist = FMT[i % 18]
    c1, c2 = off, off + 1 + dist
    b1 = ln[c1] if c1 < len(ln) else ' '
    b2 = ln[c2] if c2 < len(ln) else ' '
    assert all(ch not in 'ACGT' or j in (c1, c2) for j, ch in enumerate(ln)), (i, ln)
    if b1 in 'ACGT':
        pair = PAIR_FROM_FIRST[b1]
        assert b2 not in 'ACGT' or PAIR_FROM_SECOND[b2] == pair, (i, ln)
    elif b2 in 'ACGT':
        pair = PAIR_FROM_SECOND[b2]
    else:
        pair = None
        unknown_rungs.append(i)
    pairs.append(pair)
print('both-bases-decayed rungs:', unknown_rungs)

bits = []
for p in pairs:
    bits.extend(BITS[p] if p else '??')
raw = []
for i in range(0, len(bits) - 7, 8):
    chunk = bits[i:i + 8]
    base = sum(int(b) << k for k, b in enumerate(chunk) if b != '?')
    unk = [k for k in range(8) if chunk[k] == '?']
    raw.append((base, unk))
print('skeleton:', ''.join(chr(v) if not u and 32 <= v < 127 else '.' for v, u in raw))

CHOICES = {19: 'c', 23: 'd', 27: '_', 30: '_', 44: 's', 49: '_', 56: 'L'}
program = bytearray()
for idx, (base, unk) in enumerate(raw):
    if not unk:
        program.append(base)
        continue
    target = ord(CHOICES[idx])
    # confirm the chosen byte is inside the surviving-bit candidate set
    assert all(((target >> k) & 1) == ((base >> k) & 1) for k in range(8) if k not in unk), idx
    program.append(target)
program = bytes(program)
print('recovered program:', program)

# re-encode with the gem algorithm and compare against the handout
bitstr = ''.join(str((byte >> k) & 1) for byte in program for k in range(8))
assert len(bitstr) == 2 * len(art), (len(bitstr), len(art))
for i in range(0, len(bitstr) - 1, 2):
    off, dist = FMT[(i // 2) % 18]
    pair = PAIR_BITS[bitstr[i:i + 2]]
    rebuilt = ' ' * off + pair[0] + '-' * dist + pair[1]
    orig = art[i // 2]
    for j, ch in enumerate(orig):
        assert ch == ' ' or (j < len(rebuilt) and rebuilt[j] == ch), (i // 2, orig, rebuilt)
print('re-encode verification: PASS (every surviving character reproduced)')
print('flag:', re.findall(rb'DUCTF\{[^}]*\}', program)[0].decode())
```

Run:

```text
$ python3 dnadecay_solve.py
art lines (rungs): 236
both-bases-decayed rungs: [77, 92, 110, 111, 122, 176, 198, 227]
skeleton: puts"DUCTF{7H3_Mit0.HOn.Ri4.15.7he_P0wEr_HoU.E_of.DA_C3L.}"
recovered program: b'puts"DUCTF{7H3_Mit0cHOndRi4_15_7he_P0wEr_HoUsE_of_DA_C3LL}"'
re-encode verification: PASS (every surviving character reproduced)
flag: DUCTF{7H3_Mit0cHOndRi4_15_7he_P0wEr_HoUsE_of_DA_C3LL}
```

## Flag

```text
DUCTF{7H3_Mit0cHOndRi4_15_7he_P0wEr_HoUsE_of_DA_C3LL}
```

Sanity check: the seven hand-picked bytes are the soft spot of any recovery like this, so the script does not trust them. It re-encodes the recovered program with the gem's own algorithm and asserts a match at every position where the handout kept a non-space character, all 236 rungs. Every surviving base in the handout is reproduced exactly, so the only freedom left in the decode is inside the eight fully decayed rungs, and each of those bytes is pinned by its surviving bits plus the plain leetspeak reading.
