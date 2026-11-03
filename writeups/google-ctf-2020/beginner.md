# Beginner

- **Event:** Google CTF 2020
- **Category:** reversing
- **Difficulty:** beginner

## Summary

The handout is one small ELF binary, `a.out`. It asks for a flag, runs the input through a shuffle, an add and an xor built from SIMD instructions, and then compares the result against the input itself. The flag is a fixed point of that transform, and it also has to start with `CTF{`. I pulled the three constant tables out of the `.data` section, gave the fixed point equations to z3, and got a 15 character candidate. The binary accepted it with `SUCCESS`.

## First look

The handout came from `2020/quals/reversing-beginner/attachments/a.out` in the official google/google-ctf archive. It is a 17K, 64-bit, not stripped ELF:

```text
$ file a.out
a.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, not stripped
$ ./a.out
Flag: FAILURE
```

Running it with no useful input just prints `Flag:` and `FAILURE`, with exit code 1. The interesting strings were short:

```text
Flag:
%15s
SUCCESS
FAILURE
CTF{
```

So it reads at most 15 characters with `scanf("%15s", ...)`, and something in the binary already knows the string `CTF{`.

## What I tried first

I tried a few guesses at the prompt first, because the binary is happy to tell you no:

```text
$ echo 'CTF{test}' | ./a.out
Flag: FAILURE
$ echo 'CTF{S1MD}' | ./a.out
Flag: FAILURE
```

Then I looked at the symbol table, which gave the game away:

```text
$ nm a.out | grep -iE 'EXPECTED|SHUFFLE|ADD32'
0000000000004060 D ADD32
0000000000004080 D EXPECTED_PREFIX
0000000000004070 D SHUFFLE
```

`main` is only about 40 instructions. Reading the disassembly (`objdump -d -M intel a.out`), the flow is:

1. `scanf` the input into a 16 byte stack buffer.
2. Load it into an xmm register, byte shuffle it with `pshufb` using the `SHUFFLE` table.
3. Add the `ADD32` constants with `paddd`, four 32-bit lanes at a time.
4. Xor with a third 16 byte table at `0x4050`.
5. `strncmp(input, transformed, 16)`, then `strncmp(transformed, *EXPECTED_PREFIX, 4)`.

My first decode attempt got the lane order wrong: I treated the `ADD32` bytes as big endian dwords, and the solver came back unsatisfiable. The bytes at `0x4060` literally spell `ef be ad de ad de e1 fe 37 13 37 13 66 74 63 67` in memory, which read little endian are `0xdeadbeef`, `0xfee1dead`, `0x13371337` and `0x67637466`. That last one is the ASCII string `gctf` in reverse, a good sign the little endian reading is the intended one.

## The breakthrough

The first `strncmp` does not compare my input against a stored flag. It compares my input against its own transform. So the flag is a string `s` with `transform(s) == s`, byte for byte, plus `s` starting with `CTF{` from the second check. That is a small constraint problem, not a search problem: 15 unknown printable bytes, one NUL at position 15 because of `%15s`, and carries inside each 32-bit add lane coupling the bytes. z3 handles that directly.

The constants, from `objdump -s -j .data a.out`:

```text
Contents of section .data:
 4050 7658b449 8d1a5f38 d423f834 eb86f9aa
 4060 efbeadde addee1fe 37133713 66746367
 4070 02060701 050b090e 030f0408 0a0c0d00
 4080 20200000 00000000
```

`SHUFFLE` is the byte permutation `[2, 6, 7, 1, 5, 11, 9, 14, 3, 15, 4, 8, 10, 12, 13, 0]`, and the pointer at `0x4080` points at the `CTF{` string in `.rodata`.

## Solve

The solve script models the exact transform from `main`, one BitVec per input byte, and asks z3 for the fixed point. It then checks the answer two ways: by re-running the transform in Python, and by feeding the candidate to the real binary.

```python
import subprocess, struct
import z3

# Constants lifted from the .data section of the handout binary a.out
# (objdump -s -j .data a.out), in file order:
XOR    = bytes.fromhex("7658b4498d1a5f38d423f834eb86f9aa")
ADD32  = struct.unpack("<4I", bytes.fromhex("efbeaddeaddee1fe3713371366746367"))
SHUFFLE = [2,6,7,1,5,11,9,14,3,15,4,8,10,12,13,0]

def transform(s: bytes) -> bytes:
    shuf = bytes(s[SHUFFLE[i]] for i in range(16))
    out = bytearray(16)
    for lane in range(4):
        w = struct.unpack("<I", shuf[lane*4:lane*4+4])[0]
        w = (w + ADD32[lane]) & 0xFFFFFFFF
        wb = struct.pack("<I", w)
        for i in range(4):
            out[lane*4+i] = wb[i] ^ XOR[lane*4+i]
    return bytes(out)

s = [z3.BitVec(f"s{i}", 8) for i in range(16)]
solver = z3.Solver()
# scanf("%15s") gives 15 characters plus a NUL terminator.
solver.add(s[15] == 0)
for i in range(15):
    solver.add(s[i] >= 0x21, s[i] <= 0x7E)
# The binary compares the transformed buffer against the original input
# (strncmp(input, transformed, 16) == 0), so the flag is a fixed point.
shuf = [s[SHUFFLE[i]] for i in range(16)]
for lane in range(4):
    w = z3.Concat(shuf[lane*4+3], shuf[lane*4+2], shuf[lane*4+1], shuf[lane*4+0])
    w = w + ADD32[lane]
    for i in range(4):
        out_byte = z3.Extract(8*i+7, 8*i, w) ^ XOR[lane*4+i]
        solver.add(out_byte == s[lane*4+i])
# Second check: the transformed buffer starts with the expected prefix.
prefix = b"CTF{"
for i in range(4):
    solver.add(s[i] == prefix[i])

print("sat?", solver.check())
model = solver.model()
flag = bytes(model[x].as_long() for x in s[:15]).decode()
print("candidate:", flag)
print("transform(candidate) == candidate:", transform(flag.encode().ljust(16, b"\x00"))[:15] == flag.encode())
# Confirm against the real binary.
proc = subprocess.run(["/var/tmp/sweep2/work/beginner/a.out"], input=flag + "\n",
                      capture_output=True, text=True)
print("binary says:", proc.stdout.strip())
print(flag)
```

Output from my run:

```text
sat? sat
candidate: CTF{S1MDf0rM3!}
transform(candidate) == candidate: True
binary says: Flag: SUCCESS
CTF{S1MDf0rM3!}
```

## Flag

```text
CTF{S1MDf0rM3!}
```

Verified two ways: the Python transform of the candidate equals the candidate, and the handout binary itself prints `SUCCESS` when fed the candidate.
