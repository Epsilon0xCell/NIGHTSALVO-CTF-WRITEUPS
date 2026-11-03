# In a pickle

- **Event:** DownUnderCTF 2020
- **Category:** misc (listed as "Others" in the event program)
- **Difficulty:** easy
- **Author:** n00bmaster

## Summary

"We managed to intercept communication between und3rm4t3r and his hacker friends. However it is obfuscated using something. We just can't figure out what it is. Maybe you can help us find the flag?" The handout is a single file called `data`, 397 bytes, no extension and no format hints. It is a Python pickle, and the first move with any pickle is to disassemble it, never to load it blind.

## First look

```text
$ python3 -c "import pickletools; pickletools.dis(open('data','rb').read())" | head -14
    0: (    MARK
    1: d        DICT       (MARK at 0)
    2: p    PUT        0
    5: I    INT        1
    8: S    STRING     'D'
   13: p    PUT        1
   16: s    SETITEM
   17: I    INT        2
   20: S    STRING     'UCTF'
   28: p    PUT        2
   31: s    SETITEM
   32: I    INT        3
   35: S    STRING     '{'
   40: p    PUT        3
   43: s    SETITEM
```

The structure is obvious by the fourth entry: this is a dictionary mapping integer positions to pieces of a message. Keys 1, 2, 3 hold the strings `D`, `UCTF`, `{`, and from key 4 onward the values turn into bare integers: 112, 49, 99, 107, 108, 51, 95, 121, which are ASCII codes for `p`, `1`, `c`, `k`, `l`, `3`, `_`, `y`. The message is stored shredded by position, half as strings and half as code points.

## What I tried first

**Reaching for `pickle.load` on sight.** A pickle is a program for a stack machine, and the dangerous opcodes (`GLOBAL`, `REDUCE`, `OBJ`, `STACK_GLOBAL`) execute imports and calls during loading. Loading an unknown pickle to "see what it is" is how this challenge genre bites. The disassembly comes first: across the whole file the opcode set is only `MARK, DICT, INT, STRING, PUT, SETITEM, STOP`, pure data construction with no code path at all. Only after that check is loading it a safe act rather than a leap of faith, and the solve script asserts the opcode set before it unpickles anything.

## The breakthrough

There is no second layer. Sort the dict by key, turn integer values back into characters with `chr`, keep string values as they are, and concatenate. The pieces spell out the flag followed by a short taunt from the sender about intelligence agencies being onto him, all of it sitting in the same position map.

## Solve

```python
"""Solver: verify the pickle is data-only, then reassemble by position."""
import io, pickle, pickletools

data = open('publish/data', 'rb').read()

ops = {op.name for op, _, _ in pickletools.genops(data)}
dangerous = {'GLOBAL', 'REDUCE', 'OBJ', 'INST', 'NEWOBJ', 'EXT1', 'EXT2', 'EXT4', 'STACK_GLOBAL'}
assert not (ops & dangerous), f'executable opcodes present: {ops & dangerous}'
print('opcodes used:', sorted(ops))

d = pickle.load(io.BytesIO(data))
flag = ''.join(v if isinstance(v, str) else chr(v) for _, v in sorted(d.items()))
print('reassembled:', flag)
```

Run:

```text
$ python3 inapickle_solve.py
opcodes used: ['DICT', 'INT', 'MARK', 'PUT', 'SETITEM', 'STOP', 'STRING']
reassembled: DUCTF{p1ckl3_y0uR_m3554g3}I know that the intelligence agency's are onto me so now i'm using ways to evade them: I am just glad that you know how to use pickle. Anyway the flag is 
```

## Flag

```text
DUCTF{p1ckl3_y0uR_m3554g3}
```

Sanity check: the reassembly is not a guess at an ordering, it is the ordering the sender stored, and it produces a well-formed flag followed by clean English prose in the same map. If the position keys or the int-to-char reading were wrong anywhere, the tail of the message would garble along with the flag; instead the whole text reads end to end.
