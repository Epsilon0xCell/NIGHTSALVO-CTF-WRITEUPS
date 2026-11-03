# Symatrix

- **Event:** Google CTF 2023
- **Category:** misc
- **Difficulty:** not rated in the handout

## Summary

The challenge folder ships an encoder (`encoder.py`), an original PNG and a wider PNG called `symatrix.png` that is exactly twice as wide. The right half of the wide image is a mirror of the left half, and wherever a bit was hidden, the mirrored pixel got `(0, 1, bit)` added to it. Comparing each left pixel against its mirror and collecting the blue channel differences rebuilds the hidden bit string, which decodes to the flag in 272 bits.

## First look

The player files are in `2023/quals/misc-symatrix/challenge/` in the official google/google-ctf archive: `encoder.py`, `original.png` and `symatrix.png`. Sizes first:

```text
original size: (773, 1020)
symatrix size: (1546, 1020)
```

1546 is 2 times 773, so the encoded image carries a second copy of the picture next to the first one. The encoder source the players were given explains how that copy was made:

- `pixel_bit(b)` returns `(0, 1, b)`.
- `embed(t1, t2)` adds two pixels channel by channel.
- For each pixel at `(j, i)` in the original, the mirror position `(2 * x_len - 1 - j, i)` in the new image gets either the plain pixel, or the pixel plus `(0, 1, bit)` when a bit is being hidden there.
- Pixels whose green or blue channel is already 255 are skipped, and after each hidden bit the encoder waits a random 1 to 17 pixels before hiding the next one.
- The data being hidden is the flag file, hexlified, then turned into one long binary string.

## What I tried first

My first decoder compared the left half of `symatrix.png` against its own right half and asserted on the first differing pixel. That died immediately:

```text
AssertionError: (1, 0, (0, 0, 0, 255), (0, 0, 0))
```

Two separate problems in one line. The left half pixel came back as a 4-tuple with an alpha channel, and the mirrored pixel as a 3-tuple. The original image is RGBA and the encoded one is RGB, so raw tuple comparison is meaningless until both are converted to RGB. After `.convert("RGB")` on both images, the comparison is clean: equal pixels mean no bit, different pixels always differ by exactly `(0, 1, 0)` or `(0, 1, 1)`.

I also wasted a few minutes checking the left half of `symatrix.png` against `original.png` pixel by pixel. They match, which confirmed the left half is an untouched copy and all of the signal is in the mirror.

## The breakthrough

The encoder only ever adds `(0, 1, b)` at a hiding spot. So the green channel is a marker: a mirrored pixel with green exactly 1 higher than its source is a hiding spot, and the blue channel difference at that same spot is the bit itself, no guessing involved. The random 1 to 17 pixel gaps do not matter for decoding, because I do not need to know where the encoder chose to write. I only need to read the spots it marked, in the same row by row order it wrote them.

One conversion detail at the end: the encoder built its bit string as the binary form of the integer value of the hexlified flag, padding with a single leading zero only when the bit length was not a multiple of 8. So decoding is: bits to integer, integer to hex, hex to bytes. The recovered string was 272 bits, a multiple of 8, so no padding question came up: 272 bits is 34 bytes.

## Solve

```python
import binascii
from PIL import Image

# Player files from 2023/quals/misc-symatrix/challenge/
base = Image.open("/var/tmp/sweep2/work/symatrix/original.png").convert("RGB")
full = Image.open("/var/tmp/sweep2/work/symatrix/symatrix.png").convert("RGB")
x_len, y_len = base.size
print("original size:", base.size, "symatrix size:", full.size)
bm = base.load()
fm = full.load()
nx = x_len * 2 - 1  # encoder used nx_len = x_len*2 - 1 for the mirror column

bits = []
for i in range(y_len):
    for j in range(x_len):
        left = bm[j, i]
        right = fm[nx - j, i]
        if right != left:
            # embed() added (0, 1, bit): green always +1, blue +bit
            dg = right[1] - left[1]
            db = right[2] - left[2]
            assert dg == 1 and db in (0, 1), (j, i, left, right)
            bits.append(str(db))
print("recovered bits:", len(bits))
bitstr = "".join(bits)
# The encoder built the bit string as bin(int(hexlify(flag))) with at most
# one leading zero pad, so round the length down to a whole byte count
# that still decodes, trying the natural lengths first.
raw = None
for trim in range(0, 16):
    b = bitstr[:len(bitstr) - trim] if trim else bitstr
    if len(b) % 8 == 0:
        try:
            cand = binascii.unhexlify("%x" % int(b, 2))
            if cand.startswith(b"CTF{"):
                raw = cand
                print("trimmed", trim, "bits from the tail")
                break
        except Exception:
            pass
if raw is None:
    # Fall back: pad the recovered bits to a byte boundary.
    b = bitstr + "0" * ((-len(bitstr)) % 8)
    raw = binascii.unhexlify("%x" % int(b, 2))
print(raw.decode())
```

Output from my run:

```text
original size: (773, 1020) symatrix size: (1546, 1020)
recovered bits: 272
trimmed 0 bits from the tail
CTF{W4ke_Up_Ne0+Th1s_I5_Th3_Fl4g!}
```

## Flag

```text
CTF{W4ke_Up_Ne0+Th1s_I5_Th3_Fl4g!}
```

The decoder only reads the two player PNGs and the player encoder source. Every differing pixel matched the `(0, 1, bit)` pattern the encoder uses, with no exceptions across the whole image.
