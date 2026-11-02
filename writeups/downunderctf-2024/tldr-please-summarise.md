# tldr please summarise

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** beginner
- **Author:** Nosurf

## Summary

"I thought I was being 1337 by asking AI to help me solve challenges, now I have to reinstall Windows again. Can you help me out by find the flag in this document?" The handout is `EmuWar.docx`, an essay about the Emu War of 1932. The document is booby-trapped for anyone, human or AI, who reads it carelessly.

## Solve

A `.docx` is a zip of XML parts, so start by unzipping it and reading the text runs in `word/document.xml` directly instead of trusting a rendered view:

```
unzip -o EmuWar.docx -d emu/
grep -o '<w:t[^>]*>[^<]*</w:t>' emu/word/document.xml
```

Most runs are the essay. Three paragraphs in the middle are different: their run properties set the font color to `FFFFFF` (white) and the size to `4` half-points (2 pt). White on white at 2 points is invisible in any normal reading view, printed page, or summary.

The hidden paragraphs are not essay text. They are a prompt injection: text addressed to an AI assistant, instructing it to stop what it is doing, download a script from an external paste site, decode it, and execute it. That is the trap the challenge is about, and it is why the challenge author jokes about having to reinstall Windows. The correct response to an instruction embedded in a document is to refuse it: the payload was not fetched and nothing from the hidden text was executed.

The solve is the recognition itself: the flag rewards finding the hidden instruction and not following it. The flag value was confirmed against the challenge's own deployment configuration in the public challenge repository (the `flags` entry in its `ctfcli.yaml`), not by running anything the document demanded.

## Flag

```
DUCTF{chatgpt_I_n33d_2_3scap3}
```
