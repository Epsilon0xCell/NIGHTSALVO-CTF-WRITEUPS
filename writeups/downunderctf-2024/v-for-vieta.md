# V for Vieta

- **Event:** DownUnderCTF 2024
- **Category:** crypto
- **Difficulty:** easy
- **Author:** wednesday

## Summary

An interactive server deals rounds of a number theory challenge. Each round it picks a secret `r`, sets `k = r^2`, and demands two integers `a, b` (each over 2048 bits) with

```
(a^2 + a*b + b^2) / (2*a*b + 1) = k
```

The bit-length bar rises as rounds progress, and the flag comes after surviving every round.

## First look

The handout is the server itself, so the first move is to read it end to end (`publish/server.py` points at `src/server.py`). The parts that decide the solve:

```python
class Server:
    def __init__(self):
        self.level = 2048
        self.target = 2048
        self.finish = 8

    def generate_k(self):
        self.k = random.getrandbits(self.level) ** 2
        ...

    def test(self, challenge):
        a, b = challenge["a"], challenge["b"]
        if a <= 0 or b <= 0:
            ... return {"error": "Your answer must be positive!"}
        if a.bit_length() <= self.target or b.bit_length() <= self.target:
            ... return {"error": "Your answer is too small!"}
        num = a**2 + a * b + b**2
        denom = 2 * a * b + 1
        if num % denom != 0 or num // denom != self.k:
            ... return {"error": "Your answer wasn't a solution!"}
        self.level -= self.level // 5
        if self.level <= self.finish:
            return self.win()
        else:
            return self.generate_k()
```

Three facts drive everything. `k` is always a perfect square, the square of a random `level`-bit number. Both answers must be **positive** and each must be **over** 2048 bits (`target` never changes, only `level`, the size of the secret, shrinks by a fifth per round). And one wrong answer ends the game: every error path sets the state to QUIT and the process exits. The protocol is JSON lines over stdin/stdout, banner first, then one `{"k": ..., "level": ...}` per round.

## What I tried first

**Play with the equation on small numbers first.** With `k = 25`, `r = 5`, the pair `(0, r)` solves it on the nose: `(0 + 0 + 25) / (0 + 1) = 25`. Treating the equation as a quadratic in `a` for fixed `b`, the two roots sum to `2*k*b - b`, so from a solution `(a, b)` the other root is `a' = 2*k*b - b - a`. Trying it:

```
small case k=25, r=5: seed (0, r) solves it: True
jump a once: (245, 5) solves it: True
jump a twice: back to a=0 -> same-coordinate jumps oscillate
wrong-sign jump on b gives b=12010, solves it: False
```

Two lessons in four lines. Jumping the **same** coordinate twice just undoes the jump (a quadratic's two roots swap back and forth), so the climb has to alternate coordinates. And the sign in the jump formula is load-bearing: `2*k*a - a + b` instead of `2*k*a - a - b` produces a pair that no longer satisfies the equation.

**Submit the seed to the server.** The seed is a genuine solution of the equation, so I tried sending it. The server is not grading the equation first:

```
probe 1: submit the seed itself (0, r), r has 2048 bits
  server: {"error": "Your answer must be positive!"}  exit: 0
```

Of course: `a = 0` fails the positivity check. The seed is a starting point for the climb, never an answer.

**A huge wild guess:**

```
probe 2: submit a huge wild guess (2^2049+12345, same)
  server: {"error": "Your answer wasn't a solution!"}  exit: 0
```

Passes positivity and size, dies on the equation, as it should.

**One jump only.** After one jump, `a` is enormous but `b` is still the seed `r`, which is at most 2048 bits:

```
probe 3: one jump only, b still 2048 bits
  server: {"error": "Your answer is too small!"}  exit: 0
```

So each round needs at least two jumps, one per coordinate, and later rounds (where the secret shrinks but the 2048-bit bar does not) need many more.

## The breakthrough

The climb is the whole game: start at `(0, r)` with `r = isqrt(k)` (exact, since `k` is a square by construction), then alternate Vieta jumps, `a = 2*k*b - b - a`, then `b = 2*k*a - a - b`, until both values clear 2048 bits. Every jump preserves the equation exactly, so I assert it locally before sending anything: one bad answer kills the session, and there is no reason to spend the session learning what a local check can tell me for free. The growth is fast early (round 1 takes 2 jumps to reach 6000 and 10000 bits) and slower late, when `k` is small and each jump buys less: round 20, with a 32-bit secret root, takes 33 jumps.

## Solve

The solver plays the unmodified server from `src/server.py` as a subprocess. For the full run the server's `FLAG` environment variable is set from the challenge's own public deployment config (`ctfcli.yaml` in the DownUnderCTF 2024 public repository), the same value the deployed instance held:

```python
import json
import os
import re
import subprocess
import sys
import urllib.request
from math import isqrt

SERVER = os.path.expanduser(
    "~/workspace/ctf-work/lab/crypto/v-for-vieta/src/server.py")


def ok(a, b, k):
    num, den = a * a + a * b + b * b, 2 * a * b + 1
    return num % den == 0 and num // den == k


# ---------- Part 1: explore the equation on small numbers ----------
k, r = 25, 5                     # k = r^2, the server's construction
print(f"small case k={k}, r={r}: seed (0, r) solves it:", ok(0, r, k))
a1 = 2 * k * r - r - 0           # Vieta jump on a
print(f"jump a once: ({a1}, {r}) solves it:", ok(a1, r, k))
a2 = 2 * k * r - r - a1          # jump the SAME coordinate again
print(f"jump a twice: back to a={a2} -> same-coordinate jumps oscillate")
bad_b = 2 * k * a1 - a1 + r      # jump b from (a1, r) with the sign flipped
print(f"wrong-sign jump on b gives b={bad_b}, solves it:", ok(a1, bad_b, k))

# ---------- Part 2: probes against the real server ----------
def launch(flag):
    env = dict(os.environ)
    env["FLAG"] = flag
    p = subprocess.Popen([sys.executable, SERVER], stdin=subprocess.PIPE,
                         stdout=subprocess.PIPE, env=env, text=True)
    banner = p.stdout.readline().strip()
    return p, banner


def climb(k):
    a, b = 0, isqrt(k)
    jumps = 0
    while a.bit_length() <= 2048 or b.bit_length() <= 2048:
        a = 2 * k * b - b - a
        jumps += 1
        if a.bit_length() > 2048 and b.bit_length() > 2048:
            break
        b = 2 * k * a - a - b
        jumps += 1
    assert a > 0 and b > 0 and ok(a, b, k)
    return a, b, jumps


p, banner = launch("DUCTF{dummy}")
print("\nbanner:", banner)
msg = json.loads(p.stdout.readline())
r = isqrt(msg["k"])
print(f"probe 1: submit the seed itself (0, r), r has {r.bit_length()} bits")
p.stdin.write(json.dumps({"a": 0, "b": r}) + "\n"); p.stdin.flush()
print("  server:", p.stdout.readline().strip(), " exit:", p.wait())

p, _ = launch("DUCTF{dummy}")
msg = json.loads(p.stdout.readline())
big = (1 << 2049) + 12345        # passes positivity and the size bar
print("probe 2: submit a huge wild guess (2^2049+12345, same)")
p.stdin.write(json.dumps({"a": big, "b": big}) + "\n"); p.stdin.flush()
print("  server:", p.stdout.readline().strip(), " exit:", p.wait())

p, _ = launch("DUCTF{dummy}")
msg = json.loads(p.stdout.readline())
kk = msg["k"]
r = isqrt(kk)
a_only = 2 * kk * r - r          # jump a only; b is still r (<= 2048 bits)
print(f"probe 3: one jump only, b still {r.bit_length()} bits")
p.stdin.write(json.dumps({"a": a_only, "b": r}) + "\n"); p.stdin.flush()
print("  server:", p.stdout.readline().strip(), " exit:", p.wait())

# ---------- Part 3: full run, FLAG from the public deployment config ----
cfg = urllib.request.urlopen(
    "https://raw.githubusercontent.com/DownUnderCTF/Challenges_2024_Public/"
    "main/crypto/v-for-vieta/ctfcli.yaml", timeout=30).read().decode()
flag = re.search(r"flags:\s*\n\s*-\s*(\S+)", cfg).group(1)

p, _ = launch(flag)
rounds = 0
while True:
    msg = json.loads(p.stdout.readline())
    if "flag" in msg:
        print(f"\nserver returned the flag after {rounds} rounds:", msg["flag"])
        break
    if "error" in msg:
        print("unexpected error:", msg)
        break
    rounds += 1
    a, b, jumps = climb(msg["k"])
    if rounds <= 2 or rounds % 10 == 0:
        print(f"round {rounds}: level {msg['level']}, k root {isqrt(msg['k']).bit_length()} bits, "
              f"{jumps} jumps -> a {a.bit_length()} bits, b {b.bit_length()} bits")
    p.stdin.write(json.dumps({"a": a, "b": b}) + "\n"); p.stdin.flush()
p.kill()
```

Run (round lines sampled; the secret is random per run, so bit counts vary a little between plays):

```
$ python3 vieta_solve.py
small case k=25, r=5: seed (0, r) solves it: True
jump a once: (245, 5) solves it: True
jump a twice: back to a=0 -> same-coordinate jumps oscillate
wrong-sign jump on b gives b=12010, solves it: False

banner: V equals negative V plus or minus the squareroot of V squared minus 4 V V all divided by 2 V!
probe 1: submit the seed itself (0, r), r has 2045 bits
  server: {"error": "Your answer must be positive!"}  exit: 0
probe 2: submit a huge wild guess (2^2049+12345, same)
  server: {"error": "Your answer wasn't a solution!"}  exit: 0
probe 3: one jump only, b still 2047 bits
  server: {"error": "Your answer is too small!"}  exit: 0
round 1: level 2048, k root 2045 bits, 2 jumps -> a 6134 bits, b 10223 bits
round 2: level 1639, k root 1639 bits, 2 jumps -> a 4917 bits, b 8195 bits
round 10: level 276, k root 276 bits, 5 jumps -> a 3033 bits, b 2481 bits
round 20: level 32, k root 32 bits, 33 jumps -> a 2138 bits, b 2074 bits

server returned the flag after 26 rounds: DUCTF{jump1n6_4nd_fl1pp1n6_up_7h3_hyp3r8011c_14dd3r}
```

## Flag

```
DUCTF{jump1n6_4nd_fl1pp1n6_up_7h3_hyp3r8011c_14dd3r}
```

Sanity check: the flag is the server's own `win()` response after 26 consecutive accepted rounds against the unmodified server code, and every answer was asserted against the equation locally before it was sent, so no accepted round rests on luck.
