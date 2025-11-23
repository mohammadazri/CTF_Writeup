![header](https://capsule-render.vercel.app/api?type=waving&height=120&text=Warm%20Welcome&fontColor=00E1FF&color=0A0F1F,1A2A3A&fontSize=32&fontAlignY=40) 

<div align="center">
<picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Reverse%20Engineering-ELF%20Binary-success?logo=linux&logoColor=white&labelColor=0d1117&color=2e7d32">
    <img alt="Reverse Engineering" src="https://img.shields.io/badge/Reverse%20Engineering-ELF%20Binary-success?logo=linux&logoColor=white&labelColor=fafafa&color=2e7d32">
</picture>

<sub>Hidden 16-byte MD5 stored directly in data section → extracted and hex‑encoded to reconstruct license key.</sub>

<table>
    <tr><td><strong>CTF</strong></td><td>IGOH CTF 2025</td><td><strong>Category</strong></td><td>Reverse Engineering</td></tr>
    <tr><td><strong>Difficulty</strong></td><td>Medium</td><td><strong>Hash Type</strong></td><td>MD5 (16 bytes)</td></tr>
    <tr><td><strong>Exploit Time</strong></td><td>&lt; 3 min</td><td><strong>Flag</strong></td><td><code>igoh2025{3c07041eb3b9eccac8d4f2fe86b3df88}</code></td></tr>
</table>

<details>
<summary>Flow Diagram (Mermaid)</summary>

```mermaid
flowchart LR
    B[ELF Binary] --> R[Run binary]
    R --> P[Prompt for license]
    P --> F[Failed attempts]
    F --> RE[Open in r2]
    RE --> S[Search for 16-byte sequences]
    S --> MD5[Extract MD5 bytes]
    MD5 --> HEX[Convert to hex]
    HEX --> FLAG[Format igoh2025{md5}]
    style FLAG fill:#0b6623,stroke:#0b6623,color:#fff
```
</details>
</div>

# 🧊 Warm Welcome — Write-up

## 1. Recon

Check the binary:

```bash
file warm_welcome
```

Output:

```
ELF 64-bit LSB pie executable, not stripped
```

Running:

```bash
./warm_welcome
Enter license:
```

Any input → `invalid`.

Checking for MD5 functions:

```bash
gdb) info functions | grep -i md5
```

→ **None found**.

This suggests the binary uses a **raw MD5 digest embedded directly** rather than computing one.

---

## 2. Finding the MD5 bytes

Load in **radare2**:

```bash
r2 -AA warm_welcome
```

Locate suspicious 16-byte chunk:

```bash
s 0x2040
px 16
```

Result:

```
3c07 041e b3b9 ecca c8d4 f2fe 86b3 df88
```

This is the literal MD5 digest stored in memory.

---

## 3. Convert bytes → MD5 hex string

Raw bytes:

```
3c 07 04 1e b3 b9 ec ca c8 d4 f2 fe 86 b3 df 88
```

Convert to lowercase hex:

```
3c07041eb3b9eccac8d4f2fe86b3df88
```

This is the required license string.

---

## 4. Final Flag

```
igoh2025{3c07041eb3b9eccac8d4f2fe86b3df88}
```
