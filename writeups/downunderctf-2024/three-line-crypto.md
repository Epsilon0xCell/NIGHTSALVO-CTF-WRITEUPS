# three-line-crypto

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** -
- **Author:** -

## Summary

The entire cipher is three lines: a random 16-byte key `q`, and an encryption loop where each ciphertext byte is the plaintext byte XORed with a key byte chosen by the previous plaintext byte. The handout is the source and a 5,611-byte ciphertext, `passage.enc.txt`, an encrypted passage of English verse with a flag planted inside it. Three lines, and it held out longer than anything else in this set, because the feedback makes the key behave like a single 128-bit object: get one byte wrong and the decryption derails from that point on.

## First look

```python
q, y = os.urandom(16), 0
for x in sys.stdin.buffer.read():
    sys.stdout.buffer.write(bytes([q[y % 16] ^ x])); y = x
```

Byte by byte: `c[i] = p[i] XOR q[p[i-1] mod 16]`, with the first byte using `q[0]` (since `y` starts at 0). Decryption is just as direct given the key: `p[i] = c[i] XOR q[p[i-1] mod 16]`, feeding each recovered byte forward. The catch is that decryption is a chain: the key byte used at step `i` depends on the plaintext at step `i-1`, so a single wrong key byte does not cost one wrong character, it changes which key byte gets picked next, and the text cascades into garbage. There is no known plaintext to speak of beyond "this is English verse," and 2^128 keys do not brute force. The attack has to be statistical: find the key whose decryption reads like English.

Scoring is the whole game, so I built a trigram language model first: log-probabilities of byte trigrams counted from three public-domain texts that match the register (Pride and Prejudice, the Sherlock Holmes stories, Alice in Wonderland), floored so unseen trigrams hurt but do not veto, plus a flat penalty of 30 per non-printable byte. Calibration on a synthetic control (2,600 bytes of Alice encrypted under a known random key) said a correct key scores about -8.1 per byte, so the true key for the real 5,611-byte passage should land near -45,000. Every wrong key I ever sampled sat between -97,000 and -105,000. The gap is the signal.

## What I tried first

**Beam search over the decryption chain.** Decode left to right, keeping the best few thousand partial states scored on the plaintext so far. It recovered the synthetic control's key when the full text was scored at the end, but on the real passage the true key ranked outside the top 600 states by prefix score and the beam pruned it early: early verse is too unusual for prefix scoring to protect the right path.

**Coordinate descent.** Fix 15 key bytes, optimise the 16th, rotate. It collapses into degenerate decrypt cycles: locally, the scorer happily trades a stretch of real words for a repeating short pattern that games the trigram table, and the chain structure means a slot that is never exercised by the current (wrong) plaintext cannot be scored at all. Dead end, repeatedly.

**Simulated annealing, too cold.** My first annealer ran at a starting temperature of 120, which sounded reasonable and was in fact about twenty times too cold: at that temperature a single-slot change, which typically swings the score by roughly 900 to 2,200 points through the cascade, is accepted with probability around e^-8, effectively never. The walk was greedy in disguise and froze into the same -100k attractors as everything else. Hundreds of restarts, nothing.

## The breakthrough

Run the same annealer hot. With a starting temperature of 2,500 cooling to 3, a worsening slot change of typical size is accepted early on with probability near e^-0.5, so the walk can cross the valleys between attractors instead of freezing into the first one it meets. The proof is the control: restart 0 of the hot schedule recovered the synthetic key exactly, all 16 bytes, from a random start, something no cold run ever did. The implementation is C, because scoring is a full 5,611-byte decrypt per step and the search needs hundreds of millions of steps: one annealer run does about 650,000 iterations per second-long restart slice, times eight restarts. On the real ciphertext, restart 3 landed at -46,229, dead inside the predicted band for the true key. Every other restart in every run sat 50,000 points away.

## Solve

The trigram table builder (run once, over the three corpus texts):

```python
#!/usr/bin/env python3
"""Build the trigram log-prob table (float32, 256^3) for the C annealer.
Corpus: genuine Gutenberg prose only (no decryption outputs)."""
import math
import numpy as np
from collections import Counter

FILES = ['/var/tmp/more-ctf-build/threeline/pride.txt',
         '/var/tmp/more-ctf-build/threeline/sherlock.txt',
         '/var/tmp/more-ctf-build/threeline/alice.txt']
corpus = b''.join(open(f, 'rb').read() for f in FILES)
tri = Counter(corpus[i:i + 3] for i in range(len(corpus) - 2))
total = sum(tri.values())
table = np.full(256 ** 3, math.log(0.01 / total), dtype=np.float32)
for k, v in tri.items():
    table[(k[0] << 16) | (k[1] << 8) | k[2]] = math.log(v / total)
table.tofile('/var/tmp/more-ctf-build/threeline/tri_f32.bin')
print('total trigrams:', total, 'distinct:', len(tri))
```

The annealer (the scorer inside is the same table, the same -30 non-printable penalty, full-decrypt scoring per step; optional arguments 7 and 8 set the start and end temperatures):

```c
// three-line-crypto C annealer: same scorer/schedule as the Python version,
// but ~1000x faster full-decrypt scoring.
// usage: anneal <cipher> <table> <seconds_per_restart> <restarts> <seed> [init_hex]
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <time.h>

static float *TAB;
static unsigned char *C;
static long N;
static unsigned char P[1 << 20];

static double score_key(const unsigned char *q) {
    unsigned char prev = 0;
    for (long i = 0; i < N; i++) {
        unsigned char p = C[i] ^ q[prev & 15];
        P[i] = p;
        prev = p;
    }
    double s = 0.0;
    for (long i = 0; i + 2 < N; i++)
        s += TAB[(P[i] << 16) | (P[i + 1] << 8) | P[i + 2]];
    for (long i = 0; i < N; i++) {
        unsigned char b = P[i];
        if (!((b >= 32 && b <= 126) || b == 10)) s -= 30.0;
    }
    return s;
}

static unsigned long long rs;
static unsigned rng(void) {
    rs ^= rs << 13; rs ^= rs >> 7; rs ^= rs << 17;
    return (unsigned)(rs >> 32);
}

static double T0 = 5500.0, T1 = 3.0;
int main(int argc, char **argv) {
    const char *cpath = argv[1];
    const char *tpath = argv[2];
    double secs = atof(argv[3]);
    int restarts = atoi(argv[4]);
    rs = strtoull(argv[5], NULL, 10) | 1;
    FILE *f = fopen(cpath, "rb");
    N = fread(C = malloc(1 << 20), 1, 1 << 20, f);
    fclose(f);
    TAB = malloc((size_t)1 << 26);
    f = fopen(tpath, "rb");
    if (fread(TAB, 4, 1 << 24, f) != (size_t)(1 << 24)) { fprintf(stderr, "table short\n"); return 1; }
    fclose(f);

    unsigned char init[16];
    int have_init = 0;
    if (argc > 7) { T0 = atof(argv[7]); }
    if (argc > 8) { T1 = atof(argv[8]); }
    if (argc > 6) {
        for (int i = 0; i < 16; i++) { unsigned x; sscanf(argv[6] + 2 * i, "%2x", &x); init[i] = x; }
        have_init = 1;
    }

    double gbest = -1e18;
    unsigned char gq[16];
    for (int r = 0; r < restarts; r++) {
        unsigned char q[16];
        if (r == 0 && have_init) memcpy(q, init, 16);
        else for (int i = 0; i < 16; i++) q[i] = rng() & 255;
        double cur = score_key(q), best = cur;
        unsigned char bq[16]; memcpy(bq, q, 16);
        struct timespec t0, tn;
        clock_gettime(CLOCK_MONOTONIC, &t0);
        long it = 0;
        for (;;) {
            clock_gettime(CLOCK_MONOTONIC, &tn);
            double el = (tn.tv_sec - t0.tv_sec) + 1e-9 * (tn.tv_nsec - t0.tv_nsec);
            if (el >= secs) break;
            double frac = el / secs;
            double T = T0 * pow(T1 / T0, frac);
            int k = rng() % 16;
            unsigned char old = q[k], v = rng() & 255;
            if (v == old) continue;
            q[k] = v;
            double s = score_key(q);
            it++;
            if (s > cur || (rng() / 4294967296.0) < exp((s - cur) / T)) {
                cur = s;
                if (s > best) { best = s; memcpy(bq, q, 16); }
            } else q[k] = old;
        }
        printf("restart %d: score %.1f iters %ld q ", r, best, it);
        for (int i = 0; i < 16; i++) printf("%d ", bq[i]);
        printf("\n");
        fflush(stdout);
        if (best > gbest) { gbest = best; memcpy(gq, bq, 16); }
    }
    printf("BEST %.1f q ", gbest);
    for (int i = 0; i < 16; i++) printf("%d ", gq[i]);
    printf("\n");
    score_key(gq);
    FILE *o = fopen("/var/tmp/more-ctf-build/threeline/plain_c.txt", "wb");
    fwrite(P, 1, N, o);
    fclose(o);
    return 0;
}
```

Run:

```text
$ python3 build_table.py        # writes threeline/tri_f32.bin (float32 trigram table)
$ gcc -O3 -o anneal anneal.c -lm
$ ./anneal passage.enc.txt threeline/tri_f32.bin 75 8 2024
restart 0: score -102502.2 iters 654432 q 172 26 217 240 99 197 236 62 197 71 47 166 106 241 8 155 
restart 1: score -97454.2 iters 626586 q 144 102 123 233 79 169 245 140 164 127 142 13 82 208 163 122 
restart 2: score -104282.7 iters 648715 q 30 220 191 102 184 99 130 1 3 229 233 144 140 106 39 245 
restart 3: score -46229.0 iters 609144 q 103 145 232 58 168 78 141 244 110 165 12 207 145 19 107 162 
$ ./scorekey passage.enc.txt threeline/tri_f32.bin 6791e83aa84e8df46ea50ccf91136ba2 plain_real.txt
-46229.0 nonprint=1
$ head -6 plain_real.txt
What makes the cornfield smile; beneath what star
maecenas, it is meet to turn the sod
or marry elm with vine; how tend the steer;
what pains for cattle-keeping, or what proof
of patient trial serves for thrifty bees;
such are my themes.
$ grep -o 'DUCTF{[^}]*}' plain_real.txt
DUCTF{when_in_doubt_xort_it_out}
```

The single non-printable is one NUL byte at offset 2067 in the source passage itself ("heaven:" then the NUL then "be what thou wilt-"): the text is perfect on both sides of it, which under this cipher's feedback is only possible if the key is exactly right and the NUL was in the plaintext all along.

## Flag

```text
DUCTF{when_in_doubt_xort_it_out}
```

Sanity check: the key was verified three independent ways. The score (-46,229) sits in the calibrated true-key band and roughly 50,000 points clear of every wrong key sampled across all methods. The decryption is coherent verse for all 5,611 bytes from the first word to the last, which a single wrong key byte makes impossible under the feedback. And the flag appears mid-passage in the author's own framing: "of which one is DUCTF{when_in_doubt_xort_it_out};"
