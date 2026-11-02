# jmp flag

- **Event:** DownUnderCTF 2024
- **Category:** reversing
- **Difficulty:** easy
- **Author:** joseph

## Summary

"The flag is just a hop, a skip and a jump away." A stripped PIE binary reads up to 64 characters and prints Correct or Wrong. The flag is the 64-character input itself: there is no stored flag to extract, the input has to satisfy a machine built out of jumps.

## Solve

Disassembly and dynamic tracing show the shape of the machine:

- A 64-bit state word in `.data` starts at all ones (`0xFFFF...`).
- `main` feeds each input character through a dispatcher to a per-character handler slot; the handler for character `c` lives at `0x1304 + c * 0x80`.
- Each real handler applies one rule to the state: `if (state & gate) == 0 then state ^= toggle`, where `gate` is a small mask and `toggle` is always a single bit. In other words, a character can flip its bit only while certain other bits are still set.
- After 64 characters the binary demands the state be exactly 0.

Two traps matter. First, parsing the handlers statically is misleading: later instructions in a slot include idiom noise (an `and` with 0 on a scratch register), and some handlers use partial-register forms (`%ah`/`%al`) or a `mov $imm32` form for the toggle. The model used here was ground-truthed with a small ptrace tracer that recorded the actual state transition each handler performs, and only the first gate in each slot is real. Second, a full-length input matters: short inputs set the state to all ones (poison), and exactly 64 characters is the only way through.

With a verified model (character -> gate, toggle bit), solving is a depth-first search: find an ordering of exactly 64 effective moves that clears every bit, where a move is only legal while its gate bits are still set. One handler (`t`) toggles bit 25 unconditionally, which the search uses as a free move. The DFS finds a full clearing sequence; feeding it to the binary prints `Correct`.

```python
# model: OPS[c] = (gate_mask, toggle_bit) for each usable character c
def search(state, depth, path):
    if state == 0 and depth == 64:
        return path
    if depth == 64:
        return None
    for c, (gate, bit) in OPS.items():
        if state & gate == 0:
            r = search(state ^ (1 << bit), depth + 1, path + c)
            if r:
                return r
    return None
```

## Flag

The accepted 64-character input, wrapped as the flag:

```
DUCTF{tAb1HFK5h3ZgEX7UTMQfsivcPOaJ?nRy8jrYLVB9Ilempw6xWq2zC0d!SDukG4No}
```
