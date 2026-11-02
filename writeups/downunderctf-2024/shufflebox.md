# shufflebox

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** beginner
- **Author:** hashkitten

## Summary

"I've learned that if you shuffle your text, it's elrlay hrda to tlle htaw eht nioiglra nutpi aws." The handout shuffles text with a fixed character permutation and gives known plaintext/ciphertext pairs plus one censored line. Recover the text hidden behind the question marks and wrap it in `DUCTF{}`.

## Solve

The shuffle is a single fixed permutation of the 16 character positions, applied to every 16-character block. The handout's known pairs pin the permutation down completely: for each pair, position `i` of the plaintext lands at a fixed position of the ciphertext, and two pairs are enough to confirm every slot.

Recovered permutation (plaintext position -> ciphertext position):

```python
PERM = [9, 10, 0, 8, 11, 13, 3, 6, 15, 5, 14, 7, 4, 2, 12, 1]
```

Inverting it is then mechanical: build the inverse mapping and apply it to the censored line, whose uncensored characters agree with the known pairs at every shared position (a built-in consistency check).

```python
inv = [0] * 16
for i, p in enumerate(PERM):
    inv[p] = i
plain = ''.join(censored[inv[i]] for i in range(16))
```

The censored line unshuffles to `udiditgjwowsuper`.

## Flag

```
DUCTF{udiditgjwowsuper}
```
