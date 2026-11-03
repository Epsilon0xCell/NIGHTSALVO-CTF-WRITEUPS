# Macro Magic

- **Event:** DownUnderCTF 2024
- **Category:** forensics
- **Difficulty:** easy
- **Author:** TurboPenguin & Pix

## Summary

"We managed to pull this excel spreadsheet artifact from one of our Outpost machines. Its got something sus happening under the hood. After opening we found and captured some suspicious traffic on our network. Can you find out what this traffic is and find the flag!" The handout is a macro-bearing spreadsheet plus a packet capture of what the macro did on the wire. The work here is all static analysis: the macro is never run, and no address it names is ever contacted.

## First look

Unzipping the spreadsheet gives the usual XML parts and one `module1.bas`, the VBA module. The entry point is short enough to read in full:

```vb
Sub macro1()
    Q = "Flag: " & valueA1
    W = S + G + D + F
    O = doThing(Q, W)
    ...
End Sub
```

So the macro takes the string `"Flag: "` followed by whatever sits in cell A1 of the sheet, and runs it through `doThing` with a key `W` glued together from four variables defined elsewhere in the module: they hold `"Mon"`, `"key"`, `"Ma"` and `"gic"`, so `W = "MonkeyMagic"`. The body of `doThing` is one XOR loop:

```vb
A = A & Chr(Asc(Mid(B, I, 1)) Xor Asc(Mid(C, (I - 1) Mod Len(C) + 1, 1)))
```

That is a textbook repeating-key XOR of the input `B` against the key `C`. Two more functions finish the chain: `forensics` and `totalyFine` render the XOR output as dash-separated decimal byte values, and `superThing` sends it as the path of an HTTP GET to `http://downunderctf.com/<those numbers>`. The capture should therefore contain the encoded flag as a request path, in plain sight, if I can pick the right request out of it.

## What I tried first

**The comments in the macro.** The source is littered with comment blobs that start `MDAxMTEwMDAg...`, clearly base64. Peeling one gives a string of binary digit groups; grouping those into bytes gives decimal numbers as text: `84 114 121 32 72 97 114 100 101 114`, which is just the ASCII codes for "Try Harder". A second blob pattern decodes the same way to `70 65 75 69 ...`, the codes for `FAKEFLAG{DUCTF_Fake_Flag}`. Both are planted decoys, three layers deep, and the second one is shaped to punish anyone who grabs the first flag-shaped string they see.

**Reading the capture paths as plain ASCII.** The pcapng contains several numeric request paths, and two of them need no key at all: `80-116-101-114-111-100-97-99-116-121-108` is "Pterodactyl" as raw decimal ASCII, and a shorter run spells "Try Harder" the same way. XORing those runs with `MonkeyMagic` anyway produces junk (the solver output below shows the Pterodactyl run decoding to control characters), which is the confirmation that they never went through `doThing`: they are chaff in the same format as the real request.

## The breakthrough

The macro source fixes everything about the real request: it is the one whose bytes are `"Flag: " & flag` XORed with the repeating key `MonkeyMagic`, and the `"Flag: "` prefix is a built-in crib. So the plan is mechanical: pull every long dash-separated decimal run out of the capture bytes, XOR each with the repeating key, and keep the one that starts with `Flag: `. Exactly one run qualifies.

## Solve

```python
import re

PCAP = 'Capture.pcapng'
KEY = b'MonkeyMagic'

data = open(PCAP, 'rb').read()
runs = set()
for m in re.finditer(rb'\d+(?:-\d+){10,}', data):
    runs.add(m.group(0))
print(f'{len(runs)} candidate decimal runs in capture')

for run in sorted(runs):
    enc = bytes(int(x) for x in run.split(b'-'))
    dec = bytes(b ^ KEY[i % len(KEY)] for i, b in enumerate(enc))
    print(run[:60].decode() + '...', '->', dec[:40])
    if dec.startswith(b'Flag: '):
        print('flag:', dec[6:].decode())
```

Run:

```text
$ python3 macromagic_solve.py
2 candidate decimal runs in capture
11-3-15-12-95-89-9-52-36-61-37-54-34-90-15-86-38-26-80-19-1-... -> b'Flag: DUCTF{M4d3_W1th_AI_by_M0nk3ys}'
80-116-101-114-111-100-97-99-116-121-108... -> b'\x1d\x1b\x0b\x19\n\x1d,\x02\x13\x10\x0f'
flag: DUCTF{M4d3_W1th_AI_by_M0nk3ys}
```

## Flag

```text
DUCTF{M4d3_W1th_AI_by_M0nk3ys}
```

Sanity check: the decode is anchored at both ends. The key comes from the macro's own variable assembly, the `"Flag: "` prefix comes from the macro's own string construction, and the winning request decodes to that exact prefix followed by a well-formed flag, while the other long run in the capture fails the prefix test completely. Nothing in the solution required executing the macro or resolving the exfiltration domain: the capture already contained everything the macro sent.
