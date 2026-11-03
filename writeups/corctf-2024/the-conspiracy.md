# The Conspiracy

- **Event:** corCTF 2024
- **Category:** forensics
- **Source:** official Crusaders-of-Rust/corctf-2024-public-challenge-repo, `forensics/the-conspiracy`
- **Handouts:** `challenge.pcap`

## Summary

The capture holds a chat between three hosts, carried in TCP payloads that look like Python lists of integers. The conversation travels in pairs: one packet carries a list of large numbers, the next carries a list of small numbers between about 10 and 100, and dividing the lists element by element returns ASCII. Twenty-six messages decode into a group chat about eliminating the msfroggers, and near the end one conspirator recites the crew's secret code, which is the flag.

## First look

The pcap is small, 11,850 bytes. The global header says linktype 113, Linux cooked capture, so each packet starts with a 16 byte SLL header rather than an Ethernet header. Filtering for IPv4 and TCP leaves 52 payloads between 192.168.87.251, 192.168.134.8, and 192.168.56.19, and every payload is a printable list of integers. They alternate in a clear rhythm: a list of four and five digit numbers, then a list of two digit numbers, over and over, in both directions.

## What I tried first

My first parser assumed a standard Ethernet capture and walked the packets as if byte 14 started an IP header. It crashed inside the first few packets with an IndexError when a "TCP header" turned out to be neither aligned nor TCP: the capture also contains IPv6 and ARP frames, and the SLL framing shifts everything by two bytes compared with Ethernet. The traceback was the useful part. Checking the header's linktype field (113) explained the offset, and adding EtherType and protocol filters (IPv4 only, TCP only) cleaned the stream up.

My second wrong assumption was that the number lists might be one encoding: timestamps, a byte stream in decimal, deltas. The lists are all different lengths, which kills most stream theories, but paired lists always have the same length as each other. That pairing is the structure to exploit.

## The breakthrough

Take the first pair. Products: [6344, 1919, 3996, 7128, 9102, 1152, 4802, 8208, ...]. Keys: [61, 19, 37, 66, 82, 36, 49, 76, ...]. Dividing element by element: 6344 / 61 = 104 = 'h', 1919 / 19 = 101 = 'e', 3996 / 37 = 108 = 'l', 7128 / 66 = 108 = 'l', 9102 / 82 = 111 = 'o'. Every division comes out exact and lands in printable ASCII, so the scheme is simply character codes multiplied by small random keys, with the key list shipped in the following packet. Whatever this crew was hiding, they mailed the key under separate cover in the same capture.

## Solve

The solve script parses the pcap by hand (no tshark in the sandbox, and 52 packets do not justify one), pairs the payloads in order, and divides:

```python
import struct, ast

data = open('../handouts/challenge.pcap', 'rb').read()
assert data[:4] == b'\xd4\xc3\xb2\xa1'
linktype = struct.unpack('<I', data[20:24])[0]
assert linktype == 113, linktype   # Linux cooked capture: 16-byte SLL header
off = 24
payloads = []
while off + 16 <= len(data):
    ts_sec, ts_usec, incl, orig = struct.unpack('<IIII', data[off:off+16])
    pkt = data[off+16: off+16+incl]
    off += 16 + incl
    if len(pkt) < 16: continue
    ethertype = struct.unpack('>H', pkt[14:16])[0]
    if ethertype != 0x0800: continue          # IPv4 only
    ip = pkt[16:]
    ihl = (ip[0] & 0x0f) * 4
    if ip[9] != 6: continue                    # TCP only
    src = '.'.join(str(b) for b in ip[12:16]); dst = '.'.join(str(b) for b in ip[16:20])
    tcp = ip[ihl:]
    doff = (tcp[12] >> 4) * 4
    payload = tcp[doff:]
    if payload:
        payloads.append((src, dst, payload.decode()))
print('TCP payloads:', len(payloads))
for s, d, p in payloads: print(s, '->', d, p[:70])
for i in range(0, len(payloads) - 1, 2):
    products = ast.literal_eval(payloads[i][2])
    keys = ast.literal_eval(payloads[i+1][2])
    msg = ''.join(chr(pr // k) for pr, k in zip(products, keys))
    print('decoded:', msg)
```

Output, decoded messages (the chat rambles; the lines that matter):

```
decoded: hello blinkoid
decoded: hello night
decoded: how do we eliminate the msfroggers
decoded: idk i'll ask slice1
...
decoded: what's our code again?
decoded: i think it's corctf{b@53d_af_f0r_th3_w1n}
decoded: hey night did you hear my idea
decoded: you had an idea? blinkoid just told me you were being useless
decoded: what the sigma
```

## Flag

`corctf{b@53d_af_f0r_th3_w1n}`
