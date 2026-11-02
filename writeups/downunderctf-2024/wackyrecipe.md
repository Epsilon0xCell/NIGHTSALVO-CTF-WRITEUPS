# Wacky Recipe

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** easy

## Summary

"Our Cyber Chef has been creating some wacky recipes recently, though he has been rather protective of his secret ingredients." The handout `recipe.txt` is a program in Chef, the esoteric language where programs are recipes and values are ingredient quantities. Two ingredient values are redacted with `??`: pain and effort. Find the two values that make the recipe print the flag.

## Solve

Read the method as arithmetic. The recipe builds one number per block on the mixing bowl stack, and every block has the same shape:

1. `Put water into the mixing bowl` pushes water's value, 1.
2. `Add water` a few times: the running total is `1 + k` for `k` adds.
3. `Combine pain` multiplies the top of the stack by pain's value `P`.
4. One `Add <ingredient>` or `Remove <ingredient>` step adjusts by that ingredient's printed quantity `C` (all other quantities are given in the ingredient list: bread crumbs 4, egg yolks 13, spices 24, cheese 14, and so on).
5. `Add effort` adds effort's value `E`.

So the character built by a block is `(1 + k) * P +/- C + E`. Parsing all 20 blocks out of the recipe gives 20 such expressions in the two unknowns. The bowl is poured into the baking dish at the end, which serves the values back in reverse order, so the FIRST block produces the LAST character of the output.

Now solve against the known flag wrapper. Served first is the last block, which must produce `D`, then block 19 must produce `U`, and so on through `DUCTF{`:

```
block 20: (1 + 0) * P + 20 + E = 68    (0 adds, + sugar 20)
block 19: (1 + 1) * P + 10 + E = 85    (1 add,  + parsley 10)
```

Subtracting gives `P = 27`, and back-substituting gives `E = 21`. Cross-checks confirm it down the line (block 18: `1 * 27 + 19 + 21 = 67 = 'C'`). The full solution:

```
pain   P = 27
effort E = 21
```

Both values satisfy all 20 blocks with every result in printable range, and the served output reads as the flag:

```python
# per block: value = (1 + adds) * P (+/- C) + E ; serve in reverse block order
P, E = 27, 21
out = ''.join(chr(v) for v in reversed(block_values))
```

## Flag

```
DUCTF{2tsp_Vegemite}
```
