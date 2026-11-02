# Baby's First Forensics

- **Event:** DownUnderCTF 2024
- **Category:** forensics
- **Difficulty:** beginner
- **Author:** Pix

## Summary

"They've been trying to breach our infrastructure all morning! We need your help, we've captured some traffic of them attacking us. Can you tell us what tool they were using and its version?" The handout is a packet capture of the attack traffic.

## Solve

Open `capture.pcap` and look at the HTTP requests. The capture is a vulnerability scan: hundreds of requests probing for known paths and files, and every request carries the same User-Agent:

```
Mozilla/5.00 (Nikto/2.1.6)
```

That is the scanner identifying itself. The tool is Nikto, version 2.1.6, and the flag format is the tool name and version:

## Flag

```
DUCTF{nikto_2.1.6}
```
