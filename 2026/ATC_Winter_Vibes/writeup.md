## ATC Winter Vibes — Combined Writeup By Irza_Chad

Challenges included:
- Iced Encrypted Americano — Cryptography (25 pts)
- Thawing the Binary — Reverse (25 pts)
- Frozen Disassembly — Reverse (25 pts)
- Winter Packer — Reverse (50 pts)
- Icy Strings — Reverse (50 pts)
- My Chilly Script — Web / Client-Side Analysis (100 pts)

---

### Iced Encrypted Americano

Category: Cryptography | Points: 25

Description:
While brewing coffee at the ATC, a mysterious message appears as a sequence of hexadecimal bytes. Convert them to readable ASCII to reveal the flag.

Hex data:

```
61 74 63 63 74 66 5f 73 33 63 72 33 74 43 30 66 66 33 33
```

Solution:

Convert each hex byte to ASCII and join:

```
atcctf_s3cr3tC0ff33
```

Flag:

```
atcctf_s3cr3tC0ff33
```

Notes: basic hex→ASCII decoding; trivial but commonly-used beginner crypto exercise.

---

### Thawing the Binary

Category: Reverse | Points: 25

Challenge: Identify the type and architecture of an unknown binary without executing it.

Answer / Tool: `file` — use `file <binary>` to reveal file type and architecture (e.g., ELF 64-bit LSB executable).

---

### Frozen Disassembly

Category: Reverse | Points: 25

Challenge: Name the free, open-source reverse engineering framework that provides disassembly and decompilation (released by the NSA).

Answer: Ghidra

---

### Winter Packer

Category: Reverse | Points: 50

Challenge: Identify the common open-source executable packer/unpacker used by legitimate software and malware.

Answer: UPX

---

### Icy Strings

Category: Reverse | Points: 50

Challenge: Extract printable strings from a binary before loading it into a disassembler.

Answer / Tool: `strings` — run `strings <binary>` to enumerate embedded human-readable text.

---

### My Chilly Script

Category: Web / Client-Side Analysis | Points: 100

Overview:
An abandoned JavaScript file on the server contains multiple obfuscated representations of the same secret. Decoding any of them yields the flag.

Recon & Findings:

- Base64-encoded value: `YXRjY3RmXzBobjAzc215c2NyMXB0MXNicjBrM24=`
- Character-array with simple shifts (decimal values that map to characters after subtracting a constant)

Steps to reproduce:

1. In the browser or local editor, copy the Base64 string and decode it:

```javascript
atob('YXRjY3RmXzBobjAzc215c2NyMXB0MXNicjBrM24=')
```

2. Or, run the character-shift transformation on the numeric array:

```javascript
marshmallow.jumbo.map(n => String.fromCharCode(n - 1)).join('');
```

Decoded flag:

```
atcctf_0hn03smyscr1pt1sbr0k3n
```

Remediation:

- Never place secrets or sensitive logic in client-side code.
- Remove debug/abandoned scripts from production.
- Ensure flags are stored and validated server-side.

---

If you want, I can:

- Add a short summary table with flags, points and difficulty at the top.
- Run a spell-check and formatting pass to match your other writeups.
- Export this to a single PDF for submission.

---
