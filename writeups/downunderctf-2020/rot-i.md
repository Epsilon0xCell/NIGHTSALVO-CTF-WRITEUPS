# rot-i

- **Event:** DownUnderCTF 2020
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** joseph#8210

## Summary

"ROT13 is boring!" The handout is one line of ciphertext and the generator that made it. The generator takes a message of the form "You've solved the beginner crypto challenge! The flag is {}. ..." and encrypts it with a Caesar shift that grows as it walks the string. Recover the message, and with it the flag.

## First look

```python
from string import ascii_lowercase, ascii_uppercase

def eNCryPt(msg):
    out = ''
    for i, c in enumerate(msg):
        if c in ascii_lowercase:
            alph = ascii_lowercase
        elif c in ascii_uppercase:
            alph = ascii_uppercase
        else:
            out += c
            continue
        out += alph[(alph.index(c) + i) % len(alph)]
    return out
```

Two details decide everything. The shift for a character is its index `i` in the whole message, and `enumerate` counts every character, letters, spaces and punctuation alike. And each letter is shifted inside its own case alphabet, so case is preserved and non-letters pass through untouched. The ciphertext confirms the shape: `Ypw'zj zwufpp hwu txadjkcq dtbtyu kqkwxrbvu! Mbz cjzg kv IAJBO{ndldie_al_aqk_jjrnsxee}.` The flag-shaped token is there mid-sentence, scrambled letter by letter, and the apostrophe in `Ypw'zj` sits exactly where "You've" says it should.

## What I tried first

**Reading it as a stack of plain ROTs.** A fixed shift explains nothing here: the first word alone, `Ypw'zj`, refuses every constant shift, because the shift is 0 for `Y`, 1 for `p`, 2 for `w`, and so on. That is visible straight from the source, but the dead end worth naming is the subtler one: counting `i` over letters only. Spaces and punctuation do not get shifted, but they do advance the counter, so a letters-only count desynchronises after the first space and turns the back half of the message into noise.

## The breakthrough

There is nothing to break. The encryption is its own description of the decryption: for the character at position `i`, shift back by `i` inside the matching case alphabet, counting every character whether it moves or not. One pass, no key, no search.

## Solve

```python
from string import ascii_lowercase, ascii_uppercase

ct = open('challenge.txt').read().strip()

out = ''
for i, c in enumerate(ct):
    if c in ascii_lowercase:
        alph = ascii_lowercase
    elif c in ascii_uppercase:
        alph = ascii_uppercase
    else:
        out += c
        continue
    out += alph[(alph.index(c) - i) % len(alph)]

print(out)
import re
print('flag:', re.findall(r'DUCTF\{[^}]*\}', out))
```

Run:

```text
$ python3 roti_solve.py
You've solved the beginner crypto challenge! The flag is DUCTF{crypto_is_fun_kjqlptzy}. Now get out some pen and paper for the rest of them, they won't all be this easy :).
flag: ['DUCTF{crypto_is_fun_kjqlptzy}']
```

## Flag

```text
DUCTF{crypto_is_fun_kjqlptzy}
```

Sanity check: the whole 200-character message decodes to clean, grammatical English with correct punctuation spacing, which cannot happen by accident under a wrong counting convention: a single off-by-one in the position count garbles every letter after it.
