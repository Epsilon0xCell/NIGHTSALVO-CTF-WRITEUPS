# jmp flag

- **Event:** DownUnderCTF 2024
- **Category:** reversing
- **Difficulty:** easy
- **Author:** joseph

## Summary

"The flag is just a hop, a skip and a jump away." A stripped PIE binary reads a line of input and prints `Correct!` or `Incorrect!`. There is no stored flag to extract: the flag is the 64-character input itself, which has to satisfy a machine built out of jumps.

## First look

```
$ file publish/jmp_flag
publish/jmp_flag: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=27fc75a630e88cc3ded269794b4db7c54f9b6615, for GNU/Linux 3.2.0, stripped
$ ./publish/jmp_flag </dev/null
Incorrect!
$ printf "short" | ./publish/jmp_flag
Incorrect!
```

`strings` closes one door and opens another:

```
$ strings publish/jmp_flag | grep -i "ductf\|flag{"
Correct! DUCTF{%s}
```

There is the success message, format string and all, and no flag text anywhere: the binary wraps your own input in `DUCTF{...}` when you get it right. So the input is the flag body, and the whole problem is finding which input the machine accepts.

## What I tried first

64 copies of `A` get `Incorrect!`, as does everything else blind. The binary is 34 KB of mostly one repeated pattern, so I disassembled it (`objdump -d publish/jmp_flag`) and read the small dispatcher before the pattern starts:

```
1288:	mov    %edi,%eax
128d:	movsbq -0x4(%rbp),%rax
1292:	shl    $0x7,%rax
1296:	add    $0x60,%rax
129d:	lea    0x0(%rip),%rax        # 12a4
12a4:	add    %r11,%rax
12b3:	jmp    *%rax
```

The dispatcher takes the input character, shifts it left by 7 (multiplies by `0x80`), and jumps to `0x1304 + c * 0x80`. One 128-byte slot per byte value, 128 slots. The function just before it, at `0x1200`, reads the 64-bit word at `0x9010` in `.data` and returns whether it is zero: that word is the state, and zero is the win condition.

Slot 0 (the handler for byte `0x00`) shows the other extreme:

```
1304:	push   %rbp
1305:	mov    %rsp,%rbp
1308:	movq   $0xffffffffffffffff,0x7cfd(%rip)        # 9010
```

It resets the state to all ones. A poison slot. And a typical real handler, the slot for `A` (0x1304 + 65 * 0x80 = 0x3384):

```
3388:	mov    0x5c81(%rip),%rax        # 9010   (the state)
338f:	and    $0x2000000,%eax                   (gate: bit 25)
3394:	test   %rax,%rax
3397:	jne    33b9                                (skip if gate bit still set)
3399:	mov    0x5c70(%rip),%rax        # 9010
33a0:	xor    $0x20000,%rax                     (toggle: bit 17)
33a6:	mov    %rax,0x5c63(%rip)        # 9010
```

So each real handler is one rule: **if `(state & gate) == 0` then `state ^= toggle`**, with `toggle` a single bit. A character can flip its bit only while its gate bits are already clear. The state starts at all ones and the binary wants it at exactly zero after the input.

## The dead ends that shaped the model

**Trusting a naive static parse.** Each slot keeps going after the first gate/xor pair, and the tail is full of idiom noise: `and` instructions against scratch registers, partial-register forms (`%ah`, `%al`), `mov $imm32` variants of the constants. A parser that grabs every `and` and `xor` in a slot, or pairs a toggle with a later gate, builds a machine that does not match the binary. The parser below scans each slot in order, tracks constants as they are loaded, handles the partial-register toggle forms, and pairs every toggle with the **first** gate in its slot.

**My first exact-length search ran forever.** Knowing the rules, I wrote a depth-first search for a 64-move sequence ending at state 0, allowing any legal move. Without pruning it never came back (I killed it after a couple of minutes). The fix is arithmetic: every move flips exactly one bit, so with `rem` moves left and `pc` bits still set, finishing requires `pc <= rem` and `(rem - pc)` even. Add that prune, try the tightest-gated moves first, and the same search returns instantly, memoising zero dead states. The prune is not an optimisation detail, it is the difference between the search existing and not existing.

**Shorter is impossible, and prefixes prove nothing.** A restricted search that only allows moves which clear a set bit finds its first full clearing sequence at length exactly 64, which makes sense in hindsight: 64 bits are set, each character can clear at most one, so at least 64 characters are mathematically required. The binary agrees when fed prefixes of the real answer:

```
$ printf "tAb1HFK5h3" | ./publish/jmp_flag           (first 10 chars of the answer)
Incorrect!
$ (solution minus last char, 63 chars) | ./publish/jmp_flag
Incorrect!
$ (full 64 chars) | ./publish/jmp_flag
Correct! DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
```

## The breakthrough

The parser's census is the moment the machine becomes a puzzle instead of a binary: 64 of the 128 slots are poison (they reset the state to all ones), and the other 64 are usable moves whose toggles are all single bits, one per bit of the state. One handler stands out in the model dump: `t` has gate 0, so it fires unconditionally, the only free move on the board. From there it is a pure search problem: order exactly 64 legal moves so every bit gets cleared, where a move is legal only while its gate bits are already clear.

## Solve

```python
import re
import subprocess
import sys

ASM = open('jmpflag-fresh.asm').read().splitlines()
insn, order = {}, []
for ln in ASM:
    m = re.match(r'\s+([0-9a-f]+):\t[0-9a-f ]+\t(.*)', ln)
    if m:
        a = int(m.group(1), 16)
        insn[a] = m.group(2).strip()
        order.append(a)
order.sort()


def slot_insns(c):
    start = 0x1304 + c * 0x80
    out, a = [], start
    while a < start + 0x80 and a in insn:
        out.append(insn[a])
        if insn[a] == 'ret':
            break
        nxt = [x for x in order if a < x < start + 0x80]
        if not nxt:
            break
        a = min(nxt)
    return out


def model(c):
    """Sequential scan of one slot. Returns ('poison',) or
    ('ops', [(gate, toggle), ...]) pairing every toggle with the
    FIRST gate in the slot (later ands are idiom noise)."""
    ops = slot_insns(c)
    joined = ' | '.join(ops)
    if re.search(r'movq\s+\$0xffffffffffffffff', joined) and 'xor' not in joined:
        return ('poison',)
    seq, pending = [], None
    for t in ops:
        mm = re.match(r'movabs \$(0x[0-9a-f]+),', t)
        if mm:
            pending = int(mm.group(1), 16)
            continue
        mm = re.match(r'mov\s+\$(0x[0-9a-f]+),%e?dx', t)
        if mm:
            pending = int(mm.group(1), 16)
            continue
        if t.startswith('and') and '$' in t:
            seq.append(('gate', int(re.search(r'\$(0x[0-9a-f]+)', t).group(1), 16)))
            continue
        if re.match(r'and\s+%r', t):
            if pending is not None:
                seq.append(('gate', pending))
                pending = None
            continue
        mx = re.match(r'xor\s+\$(0x[0-9a-f]+),%(al|ah|ax|eax|rax)', t)
        if mx:
            v = int(mx.group(1), 16)
            if mx.group(2) == 'ah':
                v <<= 8
            seq.append(('toggle', v % (1 << 64)))
            continue
        if re.match(r'xor\s+%r', t):
            if pending is not None:
                seq.append(('toggle', pending))
                pending = None
            continue
    first_gate = next((v for k, v in seq if k == 'gate'), 0)
    pairs = [(first_gate, v) for k, v in seq if k == 'toggle']
    return ('ops', pairs)


OPS, POISON = {}, []
for c in range(128):
    m = model(c)
    if m[0] == 'poison':
        POISON.append(c)
    elif m[0] == 'ops' and m[1]:
        # one effective move per char in practice: keep the first pair
        OPS[c] = m[1][0]
print("poison slots (set state to all-ones):", len(POISON))
print("usable move chars:", ''.join(chr(c) for c in sorted(OPS)))
multi = {chr(c): hex(t) for c, (g, t) in OPS.items() if bin(t).count('1') != 1}
print("chars whose toggle is not a single bit:", multi)
print("sample: 'A' ->", OPS[ord('A')], " 't' ->", OPS[ord('t')])

FULL = (1 << 64) - 1
sys.setrecursionlimit(10000)


def apply(state, c):
    g, t = OPS[c]
    return state ^ t if state & g == 0 else state


def search(state, depth, target_depth, path, seen):
    if state == 0:
        return path
    if depth == target_depth:
        return None
    if (state, depth) in seen:
        return None
    seen.add((state, depth))
    for c in sorted(OPS):
        g, t = OPS[c]
        if state & g == 0 and state & t:   # only fire moves that clear a set bit
            r = search(state ^ t, depth + 1, target_depth, path + chr(c), seen)
            if r is not None:
                return r
    return None


short = search(FULL, 0, 64, '', set())
print("\nshortest clearing sequence the model finds:", len(short), "chars:", short)

# exact-64: every move flips the popcount by exactly one, so with rem moves
# left and pc bits set, a finish needs pc <= rem and (rem - pc) even.
# Try the tightest-gated legal moves first.
dead = set()
path = []


def dfs64(state, depth):
    if depth == 64:
        return state == 0
    rem = 64 - depth
    pc = bin(state).count('1')
    if pc > rem or ((rem - pc) & 1):
        return False
    if (state, depth) in dead:
        return False
    cand = []
    for c in sorted(OPS):
        g, t = OPS[c]
        if state & g == 0:
            cand.append((-bin(g).count('1'), c))
    cand.sort()
    for _, c in cand:
        path.append(chr(c))
        if dfs64(state ^ OPS[c][1], depth + 1):
            return True
        path.pop()
    dead.add((state, depth))
    return False


assert dfs64(FULL, 0)
exact = ''.join(path)
print("exact-64 sequence:", exact, " (dead states memoised:", len(dead), ")")

# emulate both through the model
for label, s in (("short", short), ("exact", exact)):
    st = FULL
    for ch in s:
        st = apply(st, ord(ch))
    print(f"model emulation of {label}: final state = {st:#x} after {len(s)} chars")

# the binary is the judge
BIN = ('/home/hatch/workspace/ctf-work/lab/rev/jmp-flag/publish/jmp_flag')
for label, s in (("short", short), ("exact", exact)):
    out = subprocess.run([BIN], input=s + '\n', capture_output=True,
                         text=True).stdout.strip()
    print(f"binary verdict on {label} ({len(s)} chars): {out}")
mut = exact[:10] + ('A' if exact[10] != 'A' else 'B') + exact[11:]
out = subprocess.run([BIN], input=mut + '\n', capture_output=True,
                     text=True).stdout.strip()
print("binary verdict on exact with char 11 mutated:", out)
print("\nflag: DUCTF{" + exact + "}")
```

Run (against a fresh `objdump -d` of the handout binary, saved as `jmpflag-fresh.asm`):

```
$ python3 jmp_solve.py
poison slots (set state to all-ones): 64
usable move chars: !0123456789?ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
chars whose toggle is not a single bit: {}
sample: 'A' -> (33554432, 131072)  't' -> (0, 33554432)

shortest clearing sequence the model finds: 64 chars: tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No
exact-64 sequence: tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No  (dead states memoised: 0 )
model emulation of short: final state = 0x0 after 64 chars
model emulation of exact: final state = 0x0 after 64 chars
binary verdict on short (64 chars): Correct! DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
binary verdict on exact (64 chars): Correct! DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
binary verdict on exact with char 11 mutated: Incorrect!

flag: DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
```

## Flag

```
DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
```

Sanity check: the model was never trusted on its own word. The sequence it produced was fed to the real binary, which printed `Correct!` and wrapped the input as the flag itself, and a one-character mutation of the same input gets `Incorrect!`, so the acceptance is the machine's and not the model's.
