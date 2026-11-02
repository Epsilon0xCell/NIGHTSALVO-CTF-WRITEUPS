# Baby's First Forensics

- **Event:** DownUnderCTF 2024
- **Category:** forensics
- **Difficulty:** beginner
- **Author:** Pix

## Summary

"They've been trying to breach our infrastructure all morning! We need your help, we've captured some traffic of them attacking us. Can you tell us what tool they were using and its version?" The handout is a packet capture of the attack traffic. The README even gives the answer format: wrap tool and version as `DUCTF{nmap_7.25}`.

## First look

The publish path is a pointer to the source copy again:

```
$ file publish/capture.pcap
publish/capture.pcap: ASCII text, with no line terminators
$ cat publish/capture.pcap
../src/capture.pcap
$ file src/capture.pcap
src/capture.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 65535)
```

The real capture is 2.6 MB. This box has no Wireshark or tshark, so the analysis below is `strings` for the first pass and a scapy script for the counting. Loading it up:

```
packets: 5747
top flows (src -> dst, packets):
  172.16.17.135 -> 172.16.17.131: 2871
  172.16.17.131 -> 172.16.17.135: 2870
```

One conversation, back and forth between two hosts. Essentially the whole capture is a single host hammering a single server.

## What I tried first

**Hunt for the flag in the traffic.** The lazy first move on any forensics handout:

```
$ strings src/capture.pcap | grep -ci DUCTF
0
```

Nothing. There is no flag in the capture to extract, which fits the question: the flag is an answer you construct, not a string you find.

**Identify the traffic by what it asks for.** The request lines are a wall of probes for known files and paths, 1,472 HTTP requests in all, and the responses say how that went:

```
first 3 requests:
  HEAD / HTTP/1.1
  GET / HTTP/1.1
  GET / HTTP/1.1
response status codes: {'404': 1417, '200': 38, '400': 10, '501': 6, '405': 2, '403': 1, '417': 1}
```

1,417 of the responses are 404. This is a vulnerability scanner working through its checklist against a server that does not have most of what it is asking for.

**The Server header, a genuine red herring.** Every response carries one:

```
Server headers (the VICTIM's software, not the attacker's): {'Apache/2.2.8 (Linux/SUSE)': 1475}
```

Tempting, because it is a product and a version sitting right there in the capture. But it answers the wrong question: Apache is the software being attacked, and the challenge asks for the tool doing the attacking.

## The breakthrough

The requests identify themselves. Pulling the `User-Agent` header out of the traffic:

```
$ strings src/capture.pcap | grep -i "user-agent" | sort | uniq -c
      1 User-Agent: Mozilla/5.00 (Nikto/2.1.6) (Evasions:None) (Test:)
      1 User-Agent: Mozilla/5.00 (Nikto/2.1.6) (Evasions:None) (Test:000001)
      1 User-Agent: Mozilla/5.00 (Nikto/2.1.6) (Evasions:None) (Test:000002)
   ... (one per test, up through the named checks: robots, sitezip,
        map_codes, embedded detection, headers: BREACH Test, and so on)
```

Every request carries the same agent with a different suffix: Nikto appends the name or number of the check it is running (`Test:000040`, `Test:sitezip`, `Test:headers: BREACH Test`). Counted raw there are 991 distinct User-Agent strings, which looks like noise until the per-test suffix is stripped. Then all 1,472 requests collapse onto a single base string.

## Solve

```python
import re
from collections import Counter

from scapy.all import IP, TCP, Raw, rdpcap

pkts = rdpcap('src/capture.pcap')
print("packets:", len(pkts))
ips = Counter()
for p in pkts:
    if IP in p:
        ips[(p[IP].src, p[IP].dst)] += 1
print("top flows (src -> dst, packets):")
for (s, d), n in ips.most_common(3):
    print(f"  {s} -> {d}: {n}")

reqs, uas, servers, statuses = [], Counter(), Counter(), Counter()
for p in pkts:
    if TCP in p and Raw in p:
        data = bytes(p[Raw]).decode(errors='ignore')
        if re.match(r'(GET|POST|HEAD|PUT|OPTIONS|PROPFIND|TRACE|TRACK|DEBUG|DELETE) ', data):
            line = data.split('\r\n', 1)[0]
            reqs.append(line)
            m = re.search(r'User-Agent: ([^\r\n]+)', data)
            if m:
                uas[m.group(1)] += 1
        if data.startswith('HTTP/'):
            statuses[data.split(' ')[1]] += 1
            m = re.search(r'Server: ([^\r\n]+)', data)
            if m:
                servers[m.group(1)] += 1

print("HTTP request lines:", len(reqs))
print("first 3 requests:")
for r in reqs[:3]:
    print(" ", r)
print("response status codes:", dict(statuses.most_common()))
print("Server headers (the VICTIM's software, not the attacker's):",
      dict(servers))

print("requests with a parseable User-Agent:", sum(uas.values()))
print("distinct full User-Agent strings among them:", len(uas),
      " (Nikto numbers its tests, and some tests repeat)")
base = Counter()
for ua, n in uas.items():
    base[re.sub(r' \(Evasions.*$', '', ua)] += n
print("User-Agents with the per-test suffix stripped:")
for ua, n in base.most_common():
    print(f"  {n:5d}  {ua}")
print("flag: DUCTF{nikto_2.1.6}  (tool lowercased, version as printed)")
```

Run:

```
$ python3 forensics_solve.py
packets: 5747
top flows (src -> dst, packets):
  172.16.17.135 -> 172.16.17.131: 2871
  172.16.17.131 -> 172.16.17.135: 2870
  172.16.17.131 -> 172.16.17.2: 2
HTTP request lines: 1472
first 3 requests:
  HEAD / HTTP/1.1
  GET / HTTP/1.1
  GET / HTTP/1.1
response status codes: {'404': 1417, '200': 38, '400': 10, '501': 6, '405': 2, '403': 1, '417': 1}
Server headers (the VICTIM's software, not the attacker's): {'Apache/2.2.8 (Linux/SUSE)': 1475}
requests with a parseable User-Agent: 1472
distinct full User-Agent strings among them: 991  (Nikto numbers its tests, and some tests repeat)
User-Agents with the per-test suffix stripped:
   1472  Mozilla/5.00 (Nikto/2.1.6)
flag: DUCTF{nikto_2.1.6}  (tool lowercased, version as printed)
```

## Flag

```
DUCTF{nikto_2.1.6}
```

Sanity check: not one request in 1,472 disagrees. With the per-test suffixes stripped, every single User-Agent in the capture is the same string, `Mozilla/5.00 (Nikto/2.1.6)`, so the tool is Nikto and the version is 2.1.6, formatted the way the README's example shows (lowercase tool name, underscore, version).
