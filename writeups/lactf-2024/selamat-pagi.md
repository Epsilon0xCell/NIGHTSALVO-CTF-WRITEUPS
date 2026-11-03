# Selamat Pagi

- **Event:** LA CTF 2024
- **Category:** crypto
- **Source:** official uclaacm/lactf-archive repo, `2024/crypto/selamat-pagi`
- **Handouts:** `message.txt`

## Summary

A monoalphabetic substitution cipher over a short Indonesian text, with the encrypted flag sitting on the third line. I never built the full key. Two cribs did the heavy lifting: the flag format fixes five letters (the cipher word before the brace must read "lactf"), and the challenge's own name fixes two whole words, because "selamat pagi" is Indonesian for "good morning" and the first two flag words match its letter patterns exactly. After the cribs, every remaining flag word had exactly one pattern-consistent match in a list of common Indonesian words, and the flag read itself out.

## First look

The message is three lines:

```
Efe kqkbkx czwkf akfs kdkf qzfskf wzdcjtfk
Ieqku kqk akfs ikxj kck akfs wkak ukikukf :Q
Lzfqztk ukdj kqk qe wefe: bkvim{wzbkdki_ckse_kckukx_ukdj_wjuk_kfkbewew_mtzujzfwe}
```

The third line ends with the giveaway shape: a word, a colon, and then `bkvim{...}`, which is the flag wearing a substitution. The flag body has seven underscore-separated words with strong internal patterns, notably `kckukx` (letters 1,3,5 repeat: pattern 1-2-1-3-1-4) and `kfkbewew` (pattern 1-2-1-3-4-5-4-5).

## What I tried first

Plain frequency analysis, the usual opener for substitution ciphers, misleads here twice over. The text is short, about 180 letters, so the counts are noisy: the cipher letter tally starts k (37), f (14), e (10), w (9). And the plaintext is Indonesian, not English, so the English reflex of mapping the top letter to 'e' is wrong; the most common letter in this text is 'a', which is exactly what k turns out to be. Frequencies alone would have walked me into a wall of plausible but wrong assignments.

## The breakthrough

Cribs beat statistics. The flag format is a crib with five letters in known positions: bkvim must decode to lactf, so b = l, k = a, v = c, i = t, m = f. The challenge title is a second crib hiding in plain sight: the event named the challenge "selamat-pagi", and the first flag word `wzbkdki` has pattern 1-2-3-4-5-4-6 while "selamat" has pattern 1-2-3-4-5-4-6. Better, the crib letters agree: selamat's third letter is l and cipher b is already l; its fourth is a and cipher k is already a; its last is t and cipher i is already t. Three independent confirmations, so the title is not a coincidence, it is the key turning. The second word `ckse` matches "pagi" the same way.

With 15 letters fixed by the cribs, each remaining flag word only needed a pattern-consistent dictionary match that did not collide with the used letters. `kckukx` could only be "apakah", `ukdj` only "kamu", `wjuk` only "suka" (and it cross-checks: its letters were already partly fixed by kamu), `kfkbewew` only "analisis", `mtzujzfwe` only "frekuensi". The partial decode of the prose lines then reads as coherent Indonesian everywhere the flag's letters reach, which is the confirmation that the key is right and not just pattern-lucky.

## Solve

```python
import re

lines = open('../handouts/selamat-pagi-message.txt').read().splitlines()
ct_flag = re.search(r'(\w+)\{([^}]*)\}', lines[2]).group(0)
print('ciphertext flag:', ct_flag)

key = {}

def add(cword, pword):
    assert len(cword) == len(pword)
    for c, p in zip(cword, pword):
        if c in key and key[c] != p:
            raise ValueError(f'conflict {c}: {key[c]} vs {p}')
        key[c] = p

def pattern(w):
    seen, out = {}, []
    for ch in w:
        if ch not in seen:
            seen[ch] = len(seen)
        out.append(seen[ch])
    return tuple(out)

# crib 1: the flag format. The cipher word before '{' must read 'lactf'.
add(ct_flag.split('{')[0], 'lactf')

# crib 2: the challenge title. 'selamat pagi' is Indonesian for
# 'good morning'; the first two flag words match its patterns.
words = ct_flag[ct_flag.index('{') + 1:-1].split('_')
assert pattern(words[0]) == pattern('selamat')
add(words[0], 'selamat')
assert pattern(words[1]) == pattern('pagi')
add(words[1], 'pagi')
print('key after cribs:', ''.join(key.get(c, '.') for c in 'abcdefghijklmnopqrstuvwxyz'))

wordlist = ('adalah pesan yang aman dengan sempurna tidak tahu apa saya katakan bendera ada sini ini dari untuk '
            'apakah kamu suka analisis frekuensi pagi selamat malam datang rumah orang waktu hari tahun kerja '
            'baik besar kecil baru lama pergi pulang makan minum baca tulis dengar lihat bicara bahasa dunia '
            'kata huruf angka rahasia sandi kunci kode teks data informasi sistem jaringan komputer aman').split()

def consistent(cw, pw):
    if pattern(cw) != pattern(pw):
        return False
    used = set(key.values())
    for c, p in zip(cw, pw):
        if c in key and key[c] != p:
            return False
        if c not in key and p in used:
            return False
    return True

for w in words[2:]:
    cands = [pw for pw in wordlist if consistent(w, pw)]
    print(w, '->', cands)
    assert len(cands) == 1
    add(w, cands[0])

flag = ''.join(key.get(ch, ch) if ch.isalpha() else ch for ch in ct_flag)
print('FLAG:', flag)
print('full decode (letters outside the flag alphabet stay unknown):')
for ln in lines:
    print(''.join(key.get(ch, '.') if ch.isalpha() else ch for ch in ln))
```

Output:

```
ciphertext flag: bkvim{wzbkdki_ckse_kckukx_ukdj_wjuk_kfkbewew_mtzujzfwe}
key after cribs: .lpmi...t.a.f.....g..cs..e
kckukx -> ['apakah']
ukdj -> ['kamu']
wjuk -> ['suka']
kfkbewew -> ['analisis']
mtzujzfwe -> ['frekuensi']
FLAG: lactf{selamat_pagi_apakah_kamu_suka_analisis_frekuensi}
full decode (letters outside the flag alphabet stay unknown):
.ni a.alah pesan .ang aman .engan sempurna
.i.ak a.a .ang tahu apa .ang sa.a katakan :.
.en.era kamu a.a .i sini: lactf{selamat_pagi_apakah_kamu_suka_analisis_frekuensi}
```

The prose decodes to readable Indonesian ("adalah pesan yang aman dengan sempurna", "tidak ada yang tahu apa yang saya katakan", "Bendera kamu ada di sini") everywhere the recovered letters reach. The letters I never recovered are the ones the flag never used, and I left them as dots rather than inventing them.

## Flag

`lactf{selamat_pagi_apakah_kamu_suka_analisis_frekuensi}`

The flag itself translates to "good morning, do you like frequency analysis". After the cribs did the work, mostly I liked patterns.
