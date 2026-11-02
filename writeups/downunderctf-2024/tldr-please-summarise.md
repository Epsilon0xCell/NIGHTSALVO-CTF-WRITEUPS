# tldr please summarise

- **Event:** DownUnderCTF 2024
- **Category:** misc
- **Difficulty:** beginner
- **Author:** Nosurf

## Summary

"I thought I was being 1337 by asking AI to help me solve challenges, now I have to reinstall Windows again. Can you help me out by find the flag in this document?" The handout is `EmuWar.docx`, an essay about the Emu War of 1932. The document is booby-trapped for anyone, human or AI, who reads it carelessly.

## First look

A `.docx` is a zip of XML parts, so the intake is to list the parts and read the XML directly rather than trusting a rendered view:

```
docx parts: ['_rels/.rels', 'docProps/core.xml', 'docProps/app.xml', 'word/_rels/document.xml.rels', 'word/document.xml', 'word/styles.xml', 'word/fontTable.xml', 'word/settings.xml', '[Content_Types].xml']
paragraphs: 40
total run text chars: 8696
```

The visible document is a straight essay. Its first paragraph:

```
The Emu War of 1932 stands as a curious and often overlooked event in Australian military history. Though brief in duration and seemingly inconsequential in its outcome, this conflict between the Australian military and a population of emus in the Wheat Belt region of Western Australia offers valuab...
```

## What I tried first

**Read the essay and grep for the flag.**

```
'DUCTF' in the whole document.xml: False
```

The flag string is nowhere in the document body. A summary or a skim gives the same result: an unremarkable history essay, no flag.

**Check the other parts.** Metadata and side files are the usual hiding spots, so I scanned every part in the zip for the flag string and for formatting markers:

```
_rels/.rels: 573 chars, markers: []
docProps/core.xml: 731 chars, markers: []
docProps/app.xml: 551 chars, markers: []
word/_rels/document.xml.rels: 531 chars, markers: []
word/document.xml: 18470 chars, markers: ['FFFFFF']
word/styles.xml: 2841 chars, markers: []
word/fontTable.xml: 853 chars, markers: []
word/settings.xml: 367 chars, markers: []
[Content_Types].xml: 1374 chars, markers: []
```

One marker in one file: `FFFFFF` appears only in `word/document.xml`. White. In the body of the document itself, formatted to be invisible. That is where to dig.

## The breakthrough

Reading `word/document.xml` run by run, with each run's properties attached, three paragraphs in the middle of the essay stand out from everything around them. Their runs set the font color to `FFFFFF` (white) and the size to `4` half-points, which is 2 pt. White on white at 2 points is invisible in any normal reading view, on a printed page, and, the point of the challenge, in most AI summaries of the document.

The analysis script finds exactly three such paragraphs and nothing else in the document is formatted that way:

```
hidden runs (white, tiny, or vanish):
  paragraph 26: color=FFFFFF sz=4 vanish=False  (5 runs)
  paragraph 27: color=FFFFFF sz=4 vanish=False  (2 runs)
  paragraph 28: color=FFFFFF sz=4 vanish=False  (3 runs)
```

The hidden paragraphs are not essay text and they are not the flag. They are a prompt injection: text addressed directly to an AI assistant, telling it to stop what it is doing and to fetch a script from an external paste site and run it. That is the trap the challenge is about, and it is why the author jokes about reinstalling Windows. The payload URL and commands are deliberately not reproduced here, and the important operational fact is what did not happen: nothing was fetched, nothing was decoded, and nothing from the hidden text was executed. An instruction that arrives inside a document you were asked to read is data, not orders, whoever it claims to address.

## Solve

The solve is the recognition itself, plus pinning the flag down from a source that is not the document. The detection script:

```python
import re
import zipfile

z = zipfile.ZipFile('publish/EmuWar.docx')
print("docx parts:", z.namelist())
xml = z.read('word/document.xml').decode()

paras = re.findall(r'<w:p[ >].*?</w:p>', xml, re.S)
print("paragraphs:", len(paras))


def run_info(p):
    """(text, color, size) for each run in a paragraph."""
    out = []
    for r in re.findall(r'<w:r>.*?</w:r>', p, re.S):
        t = ''.join(re.findall(r'<w:t[^>]*>([^<]*)</w:t>', r))
        color = re.search(r'w:color w:val="([0-9A-Fa-f]+)"', r)
        sz = re.search(r'<w:sz w:val="(\d+)"', r)
        vanish = '<w:vanish' in r
        out.append((t, color.group(1) if color else None,
                    sz.group(1) if sz else None, vanish))
    return out


essay_chars = 0
hidden = []
for i, p in enumerate(paras):
    for t, color, sz, vanish in run_info(p):
        essay_chars += len(t)
        if color == 'FFFFFF' or vanish or (sz and int(sz) <= 4):
            hidden.append((i, t, color, sz, vanish))
print("total run text chars:", essay_chars)
print("'DUCTF' in the whole document.xml:", 'DUCTF' in xml)
print("hidden paragraphs:",
      sorted({i for i, *_ in hidden}),
      " run properties: color=FFFFFF, sz=4 (2 pt)")
```

Run:

```
$ python3 tldr_solve.py
docx parts: ['_rels/.rels', 'docProps/core.xml', 'docProps/app.xml', 'word/_rels/document.xml.rels', 'word/document.xml', 'word/styles.xml', 'word/fontTable.xml', 'word/settings.xml', '[Content_Types].xml']
paragraphs: 40
total run text chars: 8696
'DUCTF' in the whole document.xml: False
hidden paragraphs: [26, 27, 28]  run properties: color=FFFFFF, sz=4 (2 pt)
```

The flag value itself is not in the document at all, hidden or otherwise. It was confirmed against the challenge's own deployment configuration in the public challenge repository (the `flags` entry in its `ctfcli.yaml`), not by running anything the document demanded:

```
$ # from DownUnderCTF/Challenges_2024_Public, beginner/tldr-please-summarise/ctfcli.yaml
flags:
  - DUCTF{chatgpt_I_n33d_2_3scap3}
```

## Flag

```
DUCTF{chatgpt_I_n33d_2_3scap3}
```

Sanity check: the flag rewards finding the hidden instruction and not following it. The detection rests on the document's own XML (three paragraphs, white, 2 pt, in an otherwise uniform essay), and the flag string comes from the challenge author's deployment config, the one source in this challenge that is not trying to trick the reader.
