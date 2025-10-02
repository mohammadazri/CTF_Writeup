# Spaghetti — Advanced Professional CTF Writeup

**Category:** Malware (John Hammond)  
**Format:** Markdown — step-by-step with reproducible commands  

---

## Executive Summary

`spaghetti` is an obfuscated PowerShell sample containing multiple embedded payloads.  
It uses `FonatozQZ("...".Replace('a','b').Replace('c','d')...)` constructs to store binary-like strings.  
After decoding, the payloads reveal:  
- AMSI patching  
- ScriptBlock logging disablement  
- Windows Defender exclusions (`C:\ProgramData\MEMEMAN\`)  

Three flags are hidden inside different decoded artifacts.  

---

## Scope & Environment

- **Goal:** recover 3 flags  
- **Environment:** isolated VM, no internet  
- **Tools:** `python3`, `ripgrep`, `strings`, `xxd`, `hexdump`  

---

## Step 1 — Initial Triage

```bash
file spaghetti
strings spaghetti | rg -i fonatozqz
```

Finds suspicious constructs:

```powershell
FonatozQZ("~%%%~~%%..." .Replace('~','0').Replace('%','1'))
```

Plan: extract → apply replaces → convert `0/1` → decode.

---

## Step 2 — Extraction Script

Save as `spaghetti_extractor.py`:

```python
#!/usr/bin/env python3
import re, sys
from pathlib import Path

inp = Path(sys.argv[1])
outd = Path("extracted_fonatozqz"); outd.mkdir(exist_ok=True)
text = inp.read_text(errors="replace")

pat = re.compile(r'FonatozQZ\("([^"]+)"(.*?)\)', re.S)
rep = re.compile(r"\.Replace\('(.+)','(.+)'\)")

def apply(s, r): 
    for a,b in rep.findall(r): s=s.replace(a,b)
    return s

for i,m in enumerate(pat.finditer(text),1):
    s=apply(m.group(1), m.group(2))
    Path(outd/f"run{i}.txt").write_text(s)
```

Run:

```bash
python3 spaghetti_extractor.py spaghetti
```

---

## Step 3 — Decode Bitstreams

Inspect outputs:

```bash
strings extracted_fonatozqz/run1.txt | head
```

Convert `0/1` to bytes (MSB first):

```python
bits="0100011001101100..."  # example
out = bytes(int(bits[i:i+8],2) for i in range(0,len(bits)//8*8,8))
print(out.decode('utf-8','ignore'))
```

If ASCII is hex or HTML entities, decode again.

---

## Step 4 — Flag 1

- **Found in:** short `FonatozQZ` literal  
- **Method:** MSB-first decode of bitstring  
- **Result:**

```
flag{39544d3b5374ebf7d39b8c260fc4afd8}
```

---

## Step 5 — Flag 2

- **Found in:** PowerShell with `HtMldECoDE('&#...;')`  
- Extract entities:

```python
import re
s="&#102;&#108;&#97;..."  
print(''.join(chr(int(n)) for n in re.findall(r'&#(\d+);', s)))
```

- **Result:**

```
flag{b313794dcef335da6206d54af81b6203}
```

---

## Step 6 — Flag 3

- **Found in:** decoded `Add-MpPreference` block  
- Search:

```bash
strings extracted_fonatozqz/run4.txt | rg "flag{"
```

- **Result:**

```
flag{60814731f508781b9a5f8636c817af9d}
```

---

## Final Flags

```
flag{39544d3b5374ebf7d39b8c260fc4afd8}
flag{b313794dcef335da6206d54af81b6203}
flag{60814731f508781b9a5f8636c817af9d}
```

---

## Indicators

- `C:\ProgramData\MEMEMAN\`  
- AMSI patch via `VirtualProtect`  
- Defender exclusions via `Add-MpPreference`  

---

## Conclusion

Obfuscation was removed by chaining Replace → bit decode → entity/hex decode.  
The sample disables detection mechanisms and hides 3 flags, all recovered using static analysis.
