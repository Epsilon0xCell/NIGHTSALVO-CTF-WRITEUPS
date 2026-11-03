# ezpz rev

- **Event:** DownUnderCTF 2022
- **Category:** reversing
- **Difficulty:** easy
- **Author:** joseph#8210

## Summary

"This challenge has a twist. Solve the puzzle, then solve the puzzle." The handout is one stripped ELF, `ezpz`, and no source. The binary hides a logic puzzle inside itself, checks a 196-character answer against four separate validators, and prints a flag file only for a grid that satisfies all of them. The reversing half is reading the puzzle out of the disassembly; the puzzle half is a Star Battle.

## First look

Strings and a quick pass over `main` set the shape: the prompt is "Enter solution to puzzle: ", failure prints "Incorrect!", and on success the program opens a file called `flag-rev.txt`, reads it and prints it. The constants do the rest. An embedded blob decodes as run-length pairs (a digit, then a letter) into a 14 by 14 map of region letters a through n, 196 cells. The input is read with `gets` into a stack buffer, so the program expects exactly the 196 characters of a grid, one per cell, and there is no length safety worth anything, but length games are not the point: the four check functions are the specification.

Reading the four validators in the disassembly gives four rules over the input grid, where a cell counts when its character is `'1'`:

1. every row contains exactly three 1s,
2. every column contains exactly three 1s,
3. every region of the decoded map contains exactly three 1s,
4. no two 1s touch, horizontally, vertically or diagonally.

That is the puzzle game Star Battle (three stars per row, column and region, no adjacency), with the region map supplied by the binary itself.

## What I tried first

**Feeding it grids by hand.** The region map prints nicely, and a human can place a few stars, but 196 cells with an exact count of three in fourteen rows, fourteen columns and fourteen regions, plus an adjacency ban, is a constraint system, not a pastime. Every manual attempt died the same way: a region filled early starved a column late. This is a solver's job.

**Greedy placement with backtracking on rows only.** My first solver draft placed row triples and checked columns at the end. It thrashed: column and region counts are global constraints, and discovering a violated column on row 13 means unwinding nearly everything. The fix is to prune with all four rules at every step: while filling a row, reject any partial row whose 1s touch the previous row's 1s (including diagonally), push the column and region counts past three, or make a count unreachable with the rows remaining. With that pruning the search finishes in about half a minute.

## The breakthrough

The real step was recognising that the binary is not hiding a flag, it is hiding a puzzle definition, and the four short validators are a complete and exact specification of it. Once the region map is decoded and the rules are transcribed, nothing about the ELF matters anymore: any grid satisfying the four counts is accepted, so the challenge reduces to one constraint solve and one run of the unmodified binary to collect the flag it prints.

## Solve

```python
"""Solver: decode the RLE region map from the binary's embedded blob,
solve the Star Battle (3 per row/col/region, no 8-neighbour contact),
then feed the grid to the real binary, which prints the flag."""
import itertools, re, subprocess, sys

BIN = 'publish/ezpz'

data = open(BIN, 'rb').read()
blob = None
for m in re.finditer(rb'(?:\d[a-z]){8,}', data):
    if len(m.group(0)) % 2 == 0:
        blob = m.group(0).decode()
        break
assert blob, 'RLE blob not found'
regions_flat = ''.join(ch * int(n) for n, ch in zip(blob[0::2], blob[1::2]))
N = 14
assert len(regions_flat) == N * N, len(regions_flat)
grid_regions = [regions_flat[r * N:(r + 1) * N] for r in range(N)]
print('region map:')
for row in grid_regions:
    print(' ', row)
region_cells = {}
for r in range(N):
    for c in range(N):
        region_cells.setdefault(grid_regions[r][c], []).append((r, c))
K = 3

row_patterns = list(itertools.combinations(range(N), K))
sol = [[0] * N for _ in range(N)]
colcount = [0] * N
regcount = {g: 0 for g in region_cells}
sys.setrecursionlimit(10000)

def solve_row(r):
    if r == N:
        return all(v == K for v in colcount) and all(v == K for v in regcount.values())
    rows_left = N - r - 1
    for pat in row_patterns:
        ok = True
        for c in pat:
            if colcount[c] >= K or colcount[c] + 1 + rows_left < K:
                ok = False; break
            g = grid_regions[r][c]
            if regcount[g] >= K:
                ok = False; break
            if r > 0:
                for dc in (-1, 0, 1):
                    cc = c + dc
                    if 0 <= cc < N and sol[r - 1][cc]:
                        ok = False; break
                if not ok: break
            for c2 in pat:
                if c2 != c and abs(c2 - c) == 1:
                    ok = False; break
            if not ok: break
        if not ok:
            continue
        for c in pat:
            sol[r][c] = 1
            colcount[c] += 1
            regcount[grid_regions[r][c]] += 1
        if solve_row(r + 1):
            return True
        for c in pat:
            sol[r][c] = 0
            colcount[c] -= 1
            regcount[grid_regions[r][c]] -= 1
    return False

assert solve_row(0), 'no solution found'
print('solution grid:')
for row in sol:
    print(' ', ''.join(map(str, row)))
answer = ''.join(str(sol[r][c]) for r in range(N) for c in range(N))
print('answer length:', len(answer))
out = subprocess.run([BIN], input=answer + '\n', capture_output=True, text=True,
                     cwd='src')  # src/ holds the binary's own flag-rev.txt
print('program output:', out.stdout.strip())
print('return code:', out.returncode)
```

Run:

```text
$ python3 ezpz_solve.py
region map:
  aaaaabbbbcccdd
  aaaaabbbbccccd
  aaeaaabbbccccd
  aaefaabbbcccgd
  eeefffffbccggd
  feeffffggggggd
  fffffffhhhgggd
  ffffhhhhhhgggd
  ffijjjjhkkllmd
  iiijjkkkkklllmm
  iijjjkkkklllmm
  iijjjkkkklllmm
  ijjjjjkknnllmm
  ijjjjnnnnnllll
solution grid:
  01010000000010
  00000101010000
  10100000000100
  00000001010001
  00010000010010
  01000001000001
  10000000010100
  00100000100010
  00001010001000
  01000000010100
  00000101000010
  10010000001000
  00101000000100
  01000001001000
program output: DUCTF{gr1d_puzzl3s_ar3_t00_ez_r1ght?}
return code: 0
```

The run happens with its working directory set to the challenge's source directory, because the binary opens `flag-rev.txt` by relative path, and the source directory is where the challenge itself keeps that file. The flag printed above is the program's own output on my computed grid: nothing was read out of the flag file directly.

## Flag

```text
DUCTF{gr1d_puzzl3s_ar3_t00_ez_r1ght?}
```

Sanity check: the flag was produced by the unmodified challenge binary accepting the grid through its own four validators and printing the file itself, which is the challenge's own success path. The grid also stands on its own: each of its rows, columns and regions contains exactly three 1s, and no two 1s are adjacent in any direction, so it satisfies the transcribed rules independent of the binary's verdict.
