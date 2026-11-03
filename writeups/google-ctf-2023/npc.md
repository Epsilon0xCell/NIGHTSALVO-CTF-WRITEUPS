# NPC

- **Event:** Google CTF 2023
- **Category:** misc
- **Difficulty:** not rated in the handout

## Summary

An age-encrypted file, `secret.age`, is locked with a password made of a few US Constitution words stuck together. The hint file, `hint.dot`, is a graph with one node per password letter: the consecutive letters of the password are edges in the graph, buried under about `len ** 1.3` random extra edges. So the password is a Hamiltonian path through the graph that happens to read as Constitution words. The graph has 28 nodes and 103 edge lines, and the full search space of word-shaped Hamiltonian paths is only 13 strings. Eleven of them were wrong; one of the 13 decrypted the file.

## First look

The player files are in `2023/quals/misc-npc/challenge/` in the official google/google-ctf archive: `encrypt.py`, `hint.dot`, `secret.age` and `USACONST.TXT`, the text of the US Constitution.

`encrypt.py` is unusually honest about its own design:

- `generate_password` builds the word list by lowercasing the Constitution, replacing every non letter with a space and splitting: 1087 distinct words by my count with the same extraction.
- The password is several of those words concatenated, chosen with `secrets.choice`.
- `generate_hint` makes one node per password letter, labelled with the letter and a random id, adds an edge for each consecutive pair of letters in the password, then adds `int(len(password) ** 1.3)` random edges between random nodes, flips some edges, and shuffles nodes and edges before writing the dot file.

Counts from the actual hint file:

```text
$ grep -c 'label=' hint.dot
28
$ grep -c -- '--' hint.dot
103
```

28 letters to find, 27 of the edges are the real path, the rest is noise. The age header in `secret.age` says the scrypt work factor is 2^19, so each password test costs a noticeable fraction of a second, but with a candidate list this small that does not matter.

## What I tried first

I wrote the search in pure Python first: depth first search over Hamiltonian paths, with a trie of the Constitution words pruning any prefix that cannot split into words, plus a connectivity check on the unused nodes. It worked, but slowly. The first candidate took about three and a half minutes to reach, and each of the first six candidates I fed to pyrage came back with a decryption failure:

```text
nodes: 28, edges: 92
candidate 1: length 28
...
candidate 6: length 28
pyrage.DecryptError: Decryption failed
```

Six word-perfect Hamiltonian paths, six wrong passwords. That is the trap built into this challenge: the dictionary test alone does not identify the password, because a 28 letter string over a 1087 word list has accidental segmentations. The only judge that counts is the age file itself.

At Python speed the remaining search was going to take longer than I wanted to wait, so I ported the exact same search to C. Nothing changed about the logic: same trie, same connectivity prune, same rule that an unused node with no way in kills the branch, same word boundary bookkeeping. It finished the entire search space in 31 seconds and printed 13 candidates total. Knowing the space is that small changes the mood completely: instead of hoping the search order gets lucky, I could test every candidate the graph allows.

## The breakthrough

The breakthrough was not a new idea, it was the count: 13. The graph plus the word list pin the password down to 13 strings, and the scrypt cost per test is small enough that trying all of them is a couple of minutes at worst. Candidates 1 to 10 failed decryption in the batch run. Candidate 11 opened the file, and its segmentation is five plain words: standard, water, sign, given, chosen.

## Solve

The full solve script is one file: it writes the C enumerator, compiles it with `gcc -O2`, runs it over `hint.dot` and `USACONST.TXT`, and feeds each candidate to `pyrage.passphrase.decrypt` until one works:

```python
#!/usr/bin/python3
"""Solve for Google CTF 2023, misc, NPC.

Player files used (official google/google-ctf archive):
  2023/quals/misc-npc/challenge/encrypt.py
  2023/quals/misc-npc/challenge/hint.dot
  2023/quals/misc-npc/challenge/secret.age
  2023/quals/misc-npc/challenge/USACONST.TXT
Local working copies are read from /var/tmp/sweep2/work/npc/.

How the challenge works (from encrypt.py): the password is a few words
from the US Constitution word list joined together. One node per letter
goes into a graph, the consecutive letter pairs of the password become
edges, then about len**1.3 extra random edges are added and everything
is shuffled into hint.dot. So the password is a Hamiltonian path through
the hint graph whose letters split into Constitution words.

This script enumerates every such path with a small C program (the same
search in pure Python took minutes per candidate; the whole space here
is only 13 candidates), then tries each password on secret.age with
pyrage until one decrypts.
"""
import subprocess
import sys
import tempfile
from pathlib import Path

BASE = Path("/var/tmp/sweep2/work/npc")

C_SOURCE = r"""
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define MAXN 64
int N; char label[MAXN]; int adj[MAXN][MAXN]; int adjlist[MAXN][MAXN];
int adjdeg[MAXN]; long ids[MAXN];
#define MAXTRIE 400000
struct TNode { int child[26]; int end; } trie[MAXTRIE];
int ntrie = 1;
void trie_insert(const char *w) {
    int cur = 0;
    for (const char *p = w; *p; p++) {
        int c = *p - 'a';
        if (!trie[cur].child[c]) trie[cur].child[c] = ntrie++;
        cur = trie[cur].child[c];
    }
    trie[cur].end = 1;
}
int used[MAXN]; int path[MAXN]; long maxcand; long emitted = 0;
void emit() {
    for (int i = 0; i < N; i++) putchar(label[path[i]]);
    putchar('\n'); fflush(stdout); emitted++;
}
int dfs(int depth, int trienode, int at_boundary) {
    if (emitted >= maxcand) return 1;
    if (depth == N) {
        if (at_boundary) { emit(); return emitted >= maxcand; }
        return 0;
    }
    int head = path[depth - 1];
    {   /* remaining graph must stay connected to the head */
        char seen[MAXN] = {0}; int stack[MAXN], sp = 0;
        stack[sp++] = head; seen[head] = 1;
        while (sp) {
            int u = stack[--sp];
            for (int k = 0; k < adjdeg[u]; k++) {
                int v = adjlist[u][k];
                if (!used[v] && !seen[v]) { seen[v] = 1; stack[sp++] = v; }
            }
        }
        for (int i = 0; i < N; i++) if (!used[i] && !seen[i]) return 0;
    }
    {   /* dead-end prune: at most one unused node may have a single way in */
        int endpoints = 0;
        for (int i = 0; i < N; i++) if (!used[i]) {
            int avail = 0;
            for (int k = 0; k < adjdeg[i]; k++) {
                int v = adjlist[i][k];
                if (!used[v] || v == head) avail++;
            }
            if (avail == 0) return 0;
            if (avail == 1) endpoints++;
        }
        if (endpoints > 1) return 0;
    }
    int cand[MAXN], cc = 0;
    for (int k = 0; k < adjdeg[head]; k++) {
        int v = adjlist[head][k];
        if (!used[v]) cand[cc++] = v;
    }
    for (int i = 0; i < cc; i++) for (int j = i + 1; j < cc; j++) {
        int di = 0, dj = 0;
        for (int k = 0; k < adjdeg[cand[i]]; k++) if (!used[adjlist[cand[i]][k]]) di++;
        for (int k = 0; k < adjdeg[cand[j]]; k++) if (!used[adjlist[cand[j]][k]]) dj++;
        if (dj < di) { int t = cand[i]; cand[i] = cand[j]; cand[j] = t; }
    }
    for (int ci = 0; ci < cc; ci++) {
        int v = cand[ci];
        int nxt = trie[trienode].child[label[v] - 'a'];
        if (!nxt) continue;
        used[v] = 1; path[depth] = v;
        if (trie[nxt].end && dfs(depth + 1, 0, 1)) return 1;
        if (dfs(depth + 1, nxt, 0)) return 1;
        used[v] = 0;
    }
    return 0;
}
int main(int argc, char **argv) {
    maxcand = atol(argv[1]);
    FILE *f = fopen(HINTDOT, "r");
    char line[512]; N = 0;
    static long edgebuf[4096][2]; int ne = 0;
    while (fgets(line, sizeof line, f)) {
        long a, b; char c;
        if (sscanf(line, " %ld [label=%c];", &a, &c) == 2) {
            ids[N] = a; label[N] = c; N++;
        } else if (sscanf(line, " %ld -- %ld;", &a, &b) == 2) {
            edgebuf[ne][0] = a; edgebuf[ne][1] = b; ne++;
        }
    }
    fclose(f);
    for (int i = 0; i < ne; i++) {
        int ia = -1, ib = -1;
        for (int j = 0; j < N; j++) {
            if (ids[j] == edgebuf[i][0]) ia = j;
            if (ids[j] == edgebuf[i][1]) ib = j;
        }
        if (ia >= 0 && ib >= 0 && !adj[ia][ib]) {
            adj[ia][ib] = adj[ib][ia] = 1;
            adjlist[ia][adjdeg[ia]++] = ib;
            adjlist[ib][adjdeg[ib]++] = ia;
        }
    }
    FILE *wf = fopen(USACONST, "rb");
    fseek(wf, 0, SEEK_END); long sz = ftell(wf); fseek(wf, 0, SEEK_SET);
    char *buf = malloc(sz + 1);
    if (fread(buf, 1, sz, wf) != (size_t)sz) return 1;
    buf[sz] = 0; fclose(wf);
    char w[128]; int wl = 0;
    for (long i = 0; i <= sz; i++) {
        unsigned char ch = (unsigned char)buf[i];
        if (ch >= 'A' && ch <= 'Z') ch += 32;
        if (ch >= 'a' && ch <= 'z') { if (wl < 120) w[wl++] = (char)ch; }
        else { if (wl) { w[wl] = 0; trie_insert(w); wl = 0; } }
    }
    fprintf(stderr, "nodes=%d edges=%d trie=%d\n", N, ne, ntrie);
    int order[MAXN];
    for (int i = 0; i < N; i++) order[i] = i;
    for (int i = 0; i < N; i++) for (int j = i + 1; j < N; j++)
        if (adjdeg[order[j]] < adjdeg[order[i]]) { int t = order[i]; order[i] = order[j]; order[j] = t; }
    for (int oi = 0; oi < N && emitted < maxcand; oi++) {
        int s = order[oi];
        int first = trie[0].child[label[s] - 'a'];
        if (!first) continue;
        memset(used, 0, sizeof used);
        used[s] = 1; path[0] = s;
        dfs(1, first, trie[first].end);
    }
    fprintf(stderr, "emitted=%ld\n", emitted);
    return 0;
}
"""


def main():
    with tempfile.TemporaryDirectory() as td:
        c_path = Path(td) / "enum.c"
        exe = Path(td) / "enum"
        src = C_SOURCE.replace("HINTDOT", f'"{BASE}/hint.dot"')
        src = src.replace("USACONST", f'"{BASE}/USACONST.TXT"')
        c_path.write_text(src)
        subprocess.run(["gcc", "-O2", "-o", str(exe), str(c_path)], check=True)
        proc = subprocess.run([str(exe), "100000"], capture_output=True, text=True)
        print(proc.stderr.strip())
        candidates = [line.strip() for line in proc.stdout.splitlines() if line.strip()]
    print(f"candidate passwords: {len(candidates)}")

    from pyrage import passphrase

    ciphertext = (BASE / "secret.age").read_bytes()
    for i, pw in enumerate(candidates, 1):
        try:
            secret = passphrase.decrypt(ciphertext, pw)
        except Exception:
            print(f"candidate {i}: decryption failed")
            continue
        print(f"candidate {i}: decryption succeeded")
        print("password:", pw)
        print("secret:", secret.decode())
        return
    sys.exit("no candidate decrypted the file")


if __name__ == "__main__":
    main()
```

Output from my run:

```text
nodes=28 edges=103 trie=3909
emitted=13
candidate passwords: 13
candidate 1: decryption failed
...
candidate 10: decryption failed
candidate 11: decryption succeeded
```

The script printed candidate 11's password and the decrypted secret right after the success line. The password is the five Constitution words from the breakthrough section joined together, and the secret is the flag below.

## Flag

```text
CTF{S3vEn_bR1dg35_0f_K0eN1g5BeRg}
```

Verified by the age decryption itself: scrypt with the recovered password and the ChaCha20-Poly1305 tag inside `secret.age` either check out or they do not, and for candidate 11 they did.
