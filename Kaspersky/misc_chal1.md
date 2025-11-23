---
title: Packet Palette CSS Color Stego
ctf: Kaspersky 2025
category: Misc / Forensics
points: 80
difficulty: Easy
date: 2025-11-23
flag: sunctf25{u6ly_c55_c0l0r5}
---

# CSS Color Steganography (Concise)

![vector](https://img.shields.io/badge/Vector-HTTP%20response%20CSS-blue) ![primitive](https://img.shields.io/badge/Primitive-color%20triplets-9c27b0) ![impact](https://img.shields.io/badge/Impact-flag%20recovery-brightgreen) ![technique](https://img.shields.io/badge/Technique-steganography-yellow) ![status](https://img.shields.io/badge/Flag-extracted-success)

## Summary
HTML in PCAP embeds sequential `color:#RRGGBB` values; interpreting each triplet’s raw bytes and filtering printable ASCII yields the flag.

## Chain
Load PCAP → extract HTML → parse CSS colors → hex→bytes → filter printable → flag.

## Recon
| Item | Observation |
|------|-------------|
| Flow | Single TCP stream to :5000 |
| Artifacts | `/dummy.js`, `/image.jpg`, `/` |
| Signal | 25 `.paragraphN` styles with unique colors |

## Extraction Steps
| Step | Action | Tool |
|------|--------|------|
| 1 | Dump traffic | `tshark -r traffic_latest.pcapng` |
| 2 | Isolate HTML | Wireshark export / tshark fields |
| 3 | Collect hex triplets | Grep `color:` |
| 4 | Convert to bytes | Python script |
| 5 | Keep printable | `32 <= b <= 126` filter |
| 6 | Output flag | Print string |

## Script
```python
colors = ["73aaee","75aaee","6eaaee","63aaee","74bb12",
          "66bb12","32bb12","35bb12","7bbb12","75ddaa",
          "36ddaa","6cddaa","79ddaa","5fddaa","63ddff",
          "35ddff","35ddff","5fddff","63ddff","30ddff",
          "6cddff","30ddff","72ddff","35ddff","7dddff"]
raw = b''.join(bytes.fromhex(x) for x in colors)
flag = ''.join(chr(b) for b in raw if 32 <= b <= 126)
print(flag)
```

## Pitfalls
| Issue | Fix |
|-------|-----|
| Extra non-printables | ASCII filter |
| Wrong order | Preserve original appearance order |
| Mixed casing | Normalize hex before parsing |

## Mitigation (Real World)
| Control | Purpose |
|---------|---------|
| Content sanitization | Strip unused style metadata |
| Traffic inspection | Detect large color sequences |
| Stego pattern rules | Alert on dense unique hex colors |

## Indicators
Unnaturally many CSS color declarations; sequential gradient-like triplets; flag-like ASCII after filtering.

## Final Flag
`sunctf25{u6ly_c55_c0l0r5}`
