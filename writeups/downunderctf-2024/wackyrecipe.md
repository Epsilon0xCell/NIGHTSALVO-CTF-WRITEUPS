# Wacky Recipe

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** easy

## Summary

"Our Cyber Chef has been creating some wacky recipes recently, though he has been rather protective of his secret ingredients." The handout `recipe.txt` is a program in Chef, the esoteric language where programs are recipes and values are ingredient quantities. Two ingredient values are redacted with `??`: pain and effort. Find the two values that make the recipe print the flag.

## First look

```
$ wc -l publish/recipe.txt
154 publish/recipe.txt

Ingredients.
?? dashes pain
?? cups effort
1 cup water
4 kg bread crumbs
26 ml hot canola oil
13 kg egg yolks
24 teaspoons all purpose spices
... (21 known quantities in all)
```

The method is 20 repetitions of one shape. The first block, verbatim:

```
Put water into 1st mixing bowl.
Add water to 1st mixing bowl.
Add water to 1st mixing bowl.
Add water to 1st mixing bowl.
Combine pain into 1st mixing bowl.
Remove bread crumbs from 1st mixing bowl.
Add effort to 1st mixing bowl.
```

And the recipe ends with:

```
Liquefy contents of the mixing bowl.
Pour contents of the mixing bowl into the 1st baking dish.
Refrigerate for 1 hour.
```

In Chef, `Put` pushes a value onto the mixing bowl stack, `Add`/`Remove` add or subtract an ingredient's quantity, `Combine` multiplies, and pouring into the baking dish serves the stack as characters. So one block builds one number, and the whole method builds 20 numbers for a 20-character output, which matches the flag wrapper plus a 14-character body.

## What I tried first

**Run it in a Chef interpreter.** There is none on this box (`which chef` finds nothing, and the two `??` values would block a straight run anyway). Just as well: executing an esolang to watch it print is the slow path when the program is this regular. Reading it as arithmetic is faster and gives the unknowns directly.

**Read the method as arithmetic, assuming the dish serves in block order.** Each block, with water = 1, `k` = the number of `Add water` lines, `P` = pain, `E` = effort, and `C` = the one adjusted ingredient's printed quantity:

1. `Put water` pushes 1, then `k` adds make the running total `1 + k`.
2. `Combine pain` multiplies by `P`.
3. One `Add <ingredient>` or `Remove <ingredient>` step adjusts by `C`.
4. `Add effort` adds `E`.

So a block builds `(1 + k) * P +/- C + E`. Parsing all 20 blocks gives 20 such expressions in two unknowns (the parse table is in the solve output below). With two unknowns, two blocks and the known flag wrapper should settle it. Assume block 1 prints first and must be `D`, block 2 prints `U`:

```
block 1: (1+3)*P - 4 + E = 68    ('D', bread crumbs 4 removed)
block 2: (1+1)*P + 26 + E = 85   ('U', hot canola oil 26 added)
```

Solving that pair gives `P = 13/2`. A non-integer pain is a contradiction: ingredient quantities in this recipe are whole numbers, and a fractional `P` would make most blocks non-printable. The equations are right and the order assumption is wrong. In Chef, pouring a stack into the baking dish serves the most recently pushed value first, so the **last** block prints the **first** character.

## The breakthrough

Flip the ends. The last block (block 20) must print `D` and block 19 must print `U`:

```
block 20: (1+0)*P + 20 + E = 68    (0 adds, + sugar 20)
block 19: (1+1)*P + 10 + E = 85    (1 add,  + parsley 10)
```

Subtracting gives `P = 27`, and back-substituting gives `E = 21`. Both whole numbers, and cross-checks confirm it down the line (block 18: `1 * 27 + 19 + 21 = 67 = 'C'`). The solve script checks all 20 blocks at once rather than trusting the first two equations.

## Solve

The parser reads the quantities from the ingredient list (handling multiword names like "all purpose spices" and skipping the two redacted ones), splits the method into blocks at each `Put water`, and records each block's add count and its single signed adjustment:

```python
import re
from fractions import Fraction

lines = open('publish/recipe.txt').read().splitlines()

# Ingredients: "<qty> <unit> <name>" or "?? <unit> <name>" for the unknowns.
qty = {}
for ln in lines:
    m = re.match(r'(\d+|\?\?) \S+ (.+)$', ln)
    if m and not ln.startswith(('Put', 'Add', 'Remove', 'Combine')):
        if m.group(1) != '??':
            qty[m.group(2)] = int(m.group(1))
print("known ingredient quantities:", len(qty),
      " unknowns: pain, effort   water =", qty['water'])

# Method: split into blocks at each 'Put water into ... mixing bowl'.
method = [ln for ln in lines if ln.startswith(
    ('Put ', 'Add ', 'Remove ', 'Combine ', 'Liquefy', 'Pour'))]
blocks, cur = [], None
for ln in method:
    if ln.startswith('Put water'):
        cur = {'adds': 0, 'adj': 0}
        blocks.append(cur)
    elif ln.startswith('Add water'):
        cur['adds'] += 1
    elif ln.startswith('Combine pain'):
        cur['combined'] = True
    elif ln.startswith('Add effort') or ln.startswith('Remove effort'):
        pass
    elif ln.startswith(('Add ', 'Remove ')):
        name = re.sub(r'^(Add|Remove) ', '', ln)
        name = re.sub(r' (to|from) .* mixing bowl\.$', '', name)
        cur['adj'] = qty[name] if ln.startswith('Add ') else -qty[name]
        cur['adj_name'] = name
print("blocks parsed:", len(blocks))
for i, b in enumerate(blocks, 1):
    print(f"  block {i:2d}: (1+{b['adds']})*P {'+' if b['adj']>=0 else '-'} "
          f"{abs(b['adj']):2d} ({b['adj_name']}) + E")


def solve(first_block_char, second_block_char, order):
    """Solve P, E from the first two served characters."""
    b1 = blocks[0] if order == 'forward' else blocks[-1]
    b2 = blocks[1] if order == 'forward' else blocks[-2]
    # (1+a1) P + E = c1 - adj1 ; (1+a2) P + E = c2 - adj2
    r1 = ord(first_block_char) - b1['adj']
    r2 = ord(second_block_char) - b2['adj']
    P = Fraction(r2 - r1, (1 + b2['adds']) - (1 + b1['adds']))
    E = r1 - (1 + b1['adds']) * P
    return P, E


# Dead end: assume the dish serves in block order (block 1 prints first).
P, E = solve('D', 'U', 'forward')
print("\nforward-order assumption: P =", P, " E =", E,
      "-> P is not an integer, contradiction")

# Reversed: the last block is served first and must print 'D', then 'U'.
P, E = solve('D', 'U', 'reversed')
print("reversed-order:           P =", P, " E =", E)

vals = [(1 + b['adds']) * int(P) + b['adj'] + int(E) for b in blocks]
out = ''.join(chr(v) for v in reversed(vals))
print("all 20 values printable:", all(32 <= v < 127 for v in vals))
print("served output:", out)
```

Run:

```
$ python3 wacky_solve.py
known ingredient quantities: 21  unknowns: pain, effort   water = 1
blocks parsed: 20
  block  1: (1+3)*P -  4 (bread crumbs) + E
  block  2: (1+1)*P + 26 (hot canola oil) + E
  block  3: (1+3)*P - 13 (egg yolks) + E
  block  4: (1+3)*P - 24 (all purpose spices) + E
  block  5: (1+2)*P +  7 (herbs) + E
  block  6: (1+1)*P + 26 (flour) + E
  block  7: (1+3)*P - 26 (sliced chicken breasts) + E
  block  8: (1+2)*P -  1 (salt) + E
  block  9: (1+1)*P + 11 (pepper) + E
  block 10: (1+2)*P -  7 (pride and joy) + E
  block 11: (1+2)*P + 10 (tomato sauce) + E
  block 12: (1+3)*P - 14 (cheese) + E
  block 13: (1+3)*P - 13 (ham) + E
  block 14: (1+0)*P +  2 (pasta sauce) + E
  block 15: (1+3)*P -  6 (chilli flakes) + E
  block 16: (1+1)*P -  5 (onion) + E
  block 17: (1+1)*P +  9 (basil) + E
  block 18: (1+0)*P + 19 (oregano) + E
  block 19: (1+1)*P + 10 (parsley) + E
  block 20: (1+0)*P + 20 (sugar) + E

forward-order assumption: P = 13/2  E = 46 -> P is not an integer, contradiction
reversed-order:           P = 27  E = 21
all 20 values printable: True
served output: DUCTF{2tsp_Vegemite}
```

The two redacted values, for the record:

```
pain   P = 27
effort E = 21
```

## Flag

```
DUCTF{2tsp_Vegemite}
```

Sanity check: `P = 27` and `E = 21` were solved from just two blocks (20 and 19 against `D` and `U`), then verified against all 20: every block produces a printable character and the served string is a well-formed flag, including the closing brace from block 1. Two equations proposed the answer, eighteen more confirmed it.
