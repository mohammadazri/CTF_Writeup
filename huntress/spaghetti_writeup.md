![header](https://capsule-render.vercel.app/api?type=waving&height=120&text=CTF%20Writeup&fontColor=00E1FF&color=0A0F1F,1A2A3A&fontSize=32&fontAlignY=40)



<div align="center">
<picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PowerShell-Obfuscation%20Cascade-critical?logo=powershell&logoColor=white&labelColor=0d1117&color=8b0000">
    <img alt="PowerShell Obfuscation Cascade" src="https://img.shields.io/badge/PowerShell-Obfuscation%20Cascade-critical?logo=powershell&logoColor=white&labelColor=fafafa&color=8b0000">
</picture>
<sub>Replace-chain expansion reveals AMSI bypass, logging suppression, Defender exclusions & three flags.</sub>

<table>
    <tr><td><strong>CTF</strong></td><td>Huntress</td><td><strong>Category</strong></td><td>Malware Reverse</td></tr>
    <tr><td><strong>Difficulty</strong></td><td>Medium</td><td><strong>Payloads</strong></td><td>Multiple FonatozQZ blocks</td></tr>
    <tr><td><strong>Flags</strong></td><td>3</td><td><strong>Time</strong></td><td>&lt; 10 min</td></tr>
</table>

<details>
  <summary><strong>▼ Flow Diagram</strong></summary>

  <div class="mermaid">
flowchart LR
  S[Script] --> R[Extract FonatozQZ blocks]
  R --> C[Replay Replace chain]
  C --> T{Type?}
  T -->|Bitstream| B[Map bits]
  T -->|Entities| E[Decode HTML]
  T -->|Plaintext| P[Search flag]
  B --> F1[Flag 1]
  E --> F2[Flag 2]
  P --> F3[Flag 3]
  style F1 fill:#0b6623,color:#fff
  style F2 fill:#0b6623,color:#fff
  style F3 fill:#0b6623,color:#fff


  </div>

</details>

</div>

# Spaghetti Multi-Flag Dropper (Concise)

![vector](https://img.shields.io/badge/Vector-obfuscated%20PowerShell-red) ![primitive](https://img.shields.io/badge/Primitive-.Replace()%20cascade-6a5acd) ![impact](https://img.shields.io/badge/Impact-flag%20extraction-critical) ![evasion](https://img.shields.io/badge/Evasion-AMSI%20patch-orange) ![status](https://img.shields.io/badge/Flags-3%20recovered-success)

## Summary
Layered `.Replace()` transforms hide AMSI bypass, logging suppression, Defender exclusions, and three flags. Replaying each cascade and decoding (bitstream, HTML entities, plaintext) reveals all artifacts.

## Chain
Identify `FonatozQZ()` blocks → extract raw literals → apply replace cascade → classify output (bits/entities/text) → decode → collect flags.

## Recon
| Item | Observation |
|------|-------------|
| Entry script | Single PowerShell dropper |
| Obfuscation | Nested `.Replace()` chains |
| Artifacts | AMSI patch, logging off, Defender exclusions |
| Flag count | 3 discrete payload runs |

## Files / Tools
| Tool | Purpose |
|------|---------|
| `strings` / `ripgrep` | Locate `FonatozQZ` markers |
| Python script | Automate cascade expansion |
| Bit/HTML decoders | Interpret transformed payloads |

## Extraction Script
```python
import re, sys, pathlib
inp = pathlib.Path(sys.argv[1])
outd = pathlib.Path('extracted_fonatozqz'); outd.mkdir(exist_ok=True)
text = inp.read_text(errors='replace')
pat = re.compile(r'FonatozQZ\("([^"]+)"(.*?)\)', re.S)
rep = re.compile(r"\.Replace\('(.+?)','(.+?)'\)")
def cascade(p, chain):
    for o,n in rep.findall(chain): p = p.replace(o,n)
    return p
for i,m in enumerate(pat.finditer(text),1):
    (outd/f'run{i}.txt').write_text(cascade(m.group(1), m.group(2)))
```

## Flag Extraction Steps
| Flag | Source | Transform | Decode | Result |
|------|--------|-----------|--------|--------|
| 1 | `run1.txt` | map `~->0` `%->1` | bits→ASCII | `flag{39544d3b...}` |
| 2 | `HtMldECoDE` block | HTML entities | regex `&#(\d+);` | `flag{b313794d...}` |
| 3 | `run4.txt` | Plaintext PS | grep `flag{` | `flag{60814731...}` |

## Decoders
```python
def bits_to_ascii(bits):
    return bytes(int(bits[i:i+8],2) for i in range(0,len(bits)//8*8,8))
def html_entities(s):
    import re; return ''.join(chr(int(n)) for n in re.findall(r'&#(\d+);', s))
```

## Indicators
| Indicator | Meaning |
|-----------|---------|
| AMSI patch via `VirtualProtect` | Script intends in‑memory AV bypass |
| ScriptBlock logging disabled | Defense telemetry suppression |
| Bulk `Add-MpPreference` exclusions | Persistence / defense weak point |
| Dense `.Replace()` chains | Obfuscation layer hiding payloads |

## Pitfalls
| Issue | Fix |
|-------|-----|
| Partial cascade replay | Apply all `.Replace()` operations in order |
| Bit mapping errors | Verify character → bit mapping before conversion |
| Missed entity decoding | Regex all `&#NNN;` occurrences |

## Mitigation
| Control | Purpose |
|---------|---------|
| AMSI integrity audit | Detect tamper attempt |
| PowerShell logging enforced | Block scriptblock disable actions |
| Defender exclusion review | Remove rogue paths |
| Obfuscation heuristics | Alert on large replace macro chains |

## Final Flags
```
flag{39544d3b5374ebf7d39b8c260fc4afd8}
flag{b313794dcef335da6206d54af81b6203}
flag{60814731f508781b9a5f8636c817af9d}
```


<div align="center">
<picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PowerShell-Obfuscation%20Cascade-critical?logo=powershell&logoColor=white&labelColor=0d1117&color=8b0000">
    <img alt="PowerShell Obfuscation Cascade" src="https://img.shields.io/badge/PowerShell-Obfuscation%20Cascade-critical?logo=powershell&logoColor=white&labelColor=fafafa&color=8b0000">
</picture>
<sub>Replace-chain expansion reveals AMSI bypass, logging suppression, Defender exclusions & three flags.</sub>

<table>
    <tr><td><strong>CTF</strong></td><td>Huntress</td><td><strong>Category</strong></td><td>Malware Reverse</td></tr>
    <tr><td><strong>Difficulty</strong></td><td>Medium</td><td><strong>Payloads</strong></td><td>Multiple FonatozQZ blocks</td></tr>
    <tr><td><strong>Flags</strong></td><td>3</td><td><strong>Time</strong></td><td>&lt; 10 min</td></tr>
</table>

<details>
  <summary><strong>▼ Flow Diagram</strong></summary>

  <div class="mermaid">
flowchart LR
  S[Script] --> R[Extract FonatozQZ blocks]
  R --> C[Replay Replace chain]
  C --> T{Type?}
  T -->|Bitstream| B[Map bits]
  T -->|Entities| E[Decode HTML]
  T -->|Plaintext| P[Search flag]
  B --> F1[Flag 1]
  E --> F2[Flag 2]
  P --> F3[Flag 3]
  style F1 fill:#0b6623,color:#fff
  style F2 fill:#0b6623,color:#fff
  style F3 fill:#0b6623,color:#fff


  </div>

</details>

</div>

# Spaghetti Multi-Flag Dropper (Concise)

![vector](https://img.shields.io/badge/Vector-obfuscated%20PowerShell-red) ![primitive](https://img.shields.io/badge/Primitive-.Replace()%20cascade-6a5acd) ![impact](https://img.shields.io/badge/Impact-flag%20extraction-critical) ![evasion](https://img.shields.io/badge/Evasion-AMSI%20patch-orange) ![status](https://img.shields.io/badge/Flags-3%20recovered-success)

## Summary
Layered `.Replace()` transforms hide AMSI bypass, logging suppression, Defender exclusions, and three flags. Replaying each cascade and decoding (bitstream, HTML entities, plaintext) reveals all artifacts.

## Chain
Identify `FonatozQZ()` blocks → extract raw literals → apply replace cascade → classify output (bits/entities/text) → decode → collect flags.

## Recon
| Item | Observation |
|------|-------------|
| Entry script | Single PowerShell dropper |
| Obfuscation | Nested `.Replace()` chains |
| Artifacts | AMSI patch, logging off, Defender exclusions |
| Flag count | 3 discrete payload runs |

## Files / Tools
| Tool | Purpose |
|------|---------|
| `strings` / `ripgrep` | Locate `FonatozQZ` markers |
| Python script | Automate cascade expansion |
| Bit/HTML decoders | Interpret transformed payloads |

## Extraction Script
```python
import re, sys, pathlib
inp = pathlib.Path(sys.argv[1])
outd = pathlib.Path('extracted_fonatozqz'); outd.mkdir(exist_ok=True)
text = inp.read_text(errors='replace')
pat = re.compile(r'FonatozQZ\("([^"]+)"(.*?)\)', re.S)
rep = re.compile(r"\.Replace\('(.+?)','(.+?)'\)")
def cascade(p, chain):
    for o,n in rep.findall(chain): p = p.replace(o,n)
    return p
for i,m in enumerate(pat.finditer(text),1):
    (outd/f'run{i}.txt').write_text(cascade(m.group(1), m.group(2)))
```

## Flag Extraction Steps
| Flag | Source | Transform | Decode | Result |
|------|--------|-----------|--------|--------|
| 1 | `run1.txt` | map `~->0` `%->1` | bits→ASCII | `flag{39544d3b...}` |
| 2 | `HtMldECoDE` block | HTML entities | regex `&#(\d+);` | `flag{b313794d...}` |
| 3 | `run4.txt` | Plaintext PS | grep `flag{` | `flag{60814731...}` |

## Decoders
```python
def bits_to_ascii(bits):
    return bytes(int(bits[i:i+8],2) for i in range(0,len(bits)//8*8,8))
def html_entities(s):
    import re; return ''.join(chr(int(n)) for n in re.findall(r'&#(\d+);', s))
```

## Indicators
| Indicator | Meaning |
|-----------|---------|
| AMSI patch via `VirtualProtect` | Script intends in‑memory AV bypass |
| ScriptBlock logging disabled | Defense telemetry suppression |
| Bulk `Add-MpPreference` exclusions | Persistence / defense weak point |
| Dense `.Replace()` chains | Obfuscation layer hiding payloads |

## Pitfalls
| Issue | Fix |
|-------|-----|
| Partial cascade replay | Apply all `.Replace()` operations in order |
| Bit mapping errors | Verify character → bit mapping before conversion |
| Missed entity decoding | Regex all `&#NNN;` occurrences |

## Mitigation
| Control | Purpose |
|---------|---------|
| AMSI integrity audit | Detect tamper attempt |
| PowerShell logging enforced | Block scriptblock disable actions |
| Defender exclusion review | Remove rogue paths |
| Obfuscation heuristics | Alert on large replace macro chains |

## Final Flags
```
flag{39544d3b5374ebf7d39b8c260fc4afd8}
flag{b313794dcef335da6206d54af81b6203}
flag{60814731f508781b9a5f8636c817af9d}
```
