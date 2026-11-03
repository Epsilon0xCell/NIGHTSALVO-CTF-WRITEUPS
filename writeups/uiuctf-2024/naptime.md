# Naptime

- **Event:** UIUCTF 2024
- **Category:** crypto
- **Author:** Anakin
- **Source:** official sigpwny/UIUCTF-2024-Public repo, `challenges/crypto/naptime`
- **Handouts:** `enc_dist.sage`, `pub.txt` (the two files the challenge lists as provided)

## Summary

A Merkle-Hellman knapsack in miniature. The cryptosystem encrypts each flag character independently as a subset sum of just eight public weights, one weight per bit of the byte. With eight weights there are only 256 subset sums in the world, so I enumerated all of them, built a sum-to-byte table, and read the 39 ciphertext values straight back into the flag. The trap is not the knapsack; it is the bit ordering, which cost me one garbage decode before I matched the generator's convention.

## First look

The distribution script shows the whole scheme. A superincreasing private sequence b is hidden by a modular multiplication (a_i = b_i * W mod M) and a permutation, giving public weights a. Encryption of one character adds up a[i] for each set bit of the character's 8 bit string. The public file gives:

```
a =  [66128, 61158, 36912, 65196, 15611, 45292, 84119, 65338]
ct = [273896, 179019, 273896, 247527, 208558, ...]
```

Thirty-nine ciphertext values, repeated values at matching positions (273896 twice at the start, 179019 at positions 1 and 7), which is what a character-by-character code should look like for a flag starting "uiuctf{".

Classical knapsack attacks (recovering b, W, and M, or running a lattice reduction) exist because real Merkle-Hellman uses hundreds of weights. Here n = 8 per character. The subset structure is tiny enough to enumerate outright.

## What I tried first

My first solver enumerated the 256 subsets, keyed the table by subset sum, and decoded. It produced mostly empty lookups and, where it did hit, nonsense: fragments like `.f`, `6`, `,` at scattered positions. Two hypotheses followed. First, that the ciphertext had been reduced modulo the secret M, so each value needed a multiple of M added back before it became a subset sum. I searched for an M that made every ciphertext value congruent to some subset sum, and hundreds of spurious small moduli "worked" while decoding to garbage, which is the signature of a theory with too many free parameters.

The second look killed both theories: a direct membership test showed all 39 ciphertext values are exact subset sums of a, no modulus involved. The bug was in my table, not the cipher. I had built each byte as if the first weight corresponded to the least significant bit, but the generator's bit string runs most significant bit first: position i of the string selects a[i] and carries value 2^(7-i). My first decode had been reading every byte bit-reversed, which is why it produced plausible-looking punctuation instead of text.

## The breakthrough

Once the subset sums were confirmed present, the solve is a lookup table with the generator's own bit convention: for each subset, the byte value is the sum of 2^(7-i) over the chosen indices. All 256 subset sums turn out to be distinct, so the table is a perfect codebook and every ciphertext value decodes to exactly one byte.

## Solve

```python
import re, itertools

txt = open('../handouts/pub.txt').read()
a = [int(x) for x in re.search(r'a =  \[([^\]]*)\]', txt).group(1).split(',')]
ct = [int(x) for x in re.search(r'ct = \[([^\]]*)\]', txt).group(1).split(',')]

# every subset sum of the 8 public weights -> the byte it encodes.
# in enc(), bit position i of the 8-bit string (MSB first) selects a[i];
# that bit carries value 2**(7-i).
table = {}
for r in range(9):
    for combo in itertools.combinations(range(8), r):
        s = sum(a[i] for i in combo)
        table[s] = sum(2 ** (7 - i) for i in combo)
print('distinct subset sums:', len(table))
flag = bytes(table[v] for v in ct)
print(flag.decode())
```

Output:

```
distinct subset sums: 256
uiuctf{i_g0t_sleepy_s0_I_13f7_th3_fl4g}
```

## Flag

`uiuctf{i_g0t_sleepy_s0_I_13f7_th3_fl4g}`
