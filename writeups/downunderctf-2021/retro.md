# Retro

- **Event:** DownUnderCTF 2021
- **Category:** misc (listed under forensics in the repo)
- **Difficulty:** easy
- **Author:** Yo_Yo_Bro
- **Source:** official DownUnderCTF/Challenges_2021_Public repo, `forensics/retro`
- **Handouts:** `og.jpg`

## Summary

The challenge hands over the event's original logo, "created in paint", and asks what other secrets it hides. The answer is sitting in the image's XMP metadata: the Dublin Core creator field carries the flag string in plain text.

## First look

The file is a 123,612 byte JPEG. The README gives one unusual instruction: the flag format is "STRING you end up with after solving challenge", which hints the payload might not wear the usual DUCTF wrapper, so I went in expecting arbitrary text rather than scanning for the prefix alone. That mattered, because a prefix-only grep mindset is exactly how a metadata field gets missed.

## What I tried first

The standard first pass for a JPEG is looking for data appended after the end-of-image marker, since that is where lazy steganography lives. I measured it directly: the FF D9 marker sits at the very end of the file, with 0 bytes after it. No appended zip, no trailing second image, nothing.

My second pass checked the usual EXIF fields, and came back empty as well. A Paint-era logo with cleaned or absent EXIF is normal, so at that point the file looked boring from every angle except one: JPEGs can carry an XMP packet in an APP1 segment, and metadata editors love writing creator fields there.

## The breakthrough

A printable-strings sweep over the whole file, not just the metadata region, surfaced an RDF block at offset 2150 containing a `dc:creator` element whose list item is a complete flag string, repeated in the raw XMP and visible in the strings output. The image pixels were never involved. Paint did not hide the secret; the file format's paperwork did.

## Solve

Rather than trusting the strings hit, the solve script walks the JPEG marker segments properly, reports each segment, and extracts the creator value out of the XMP packet with a targeted match:

```python
import re, struct

d = open('../handouts/og.jpg', 'rb').read()
print('file size:', len(d))
eoi = d.rfind(b'\xff\xd9')
print('bytes after JPEG EOI marker:', len(d) - (eoi + 2))

# walk the JPEG marker segments looking for an XMP packet
off = 2
flag = None
while off + 4 <= len(d) and d[off] == 0xff:
    marker = d[off + 1]
    if marker in (0xd8, 0xd9) or 0xd0 <= marker <= 0xd7:
        off += 2
        continue
    seglen = struct.unpack('>H', d[off + 2:off + 4])[0]
    seg = d[off + 4: off + 2 + seglen]
    print(f'segment FF{marker:02X} at {off}, {seglen} bytes:', seg[:40])
    if b'ns.adobe.com/xap/1.0' in seg:
        m = re.search(rb'<rdf:li>([^<]+)</rdf:li>', seg)
        if m:
            flag = m.group(1).decode()
    if marker == 0xda:
        break
    off += 2 + seglen

print('XMP dc:creator:', flag)
```

Output (segment list trimmed to the lines that matter):

```
bytes after JPEG EOI marker: 0
...
XMP dc:creator: DUCTF{sicc_paint_skillz!}
```

## Flag

`DUCTF{sicc_paint_skillz!}`
