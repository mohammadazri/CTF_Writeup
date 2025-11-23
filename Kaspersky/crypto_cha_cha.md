---
title: Resemblance ChaCha20 Keystream Reuse
ctf: Kaspersky 2025
category: Crypto
points: 150
difficulty: Medium
date: 2025-11-23
flag: sunctf25{m4yb3_s0m3_k3y_d1ff3r3nc3_1snt_s0_b4d_4ft3r4LL}
---

# ChaCha20 Keystream Reuse (Concise)

![vector](https://img.shields.io/badge/Vector-keystream%20reuse-red) ![primitive](https://img.shields.io/badge/Primitive-ChaCha20-0a84ff) ![impact](https://img.shields.io/badge/Impact-flag%20recovery-critical) ![mode](https://img.shields.io/badge/Mode-stream-purple) ![status](https://img.shields.io/badge/Flag-extracted-success)

## Summary
Known plaintext + reused (key, nonce) pair → identical keystream for story and flag → XOR to recover keystream then flag.

## Chain
Parse hex → derive keystream (`C_msg ⊕ P_msg`) → XOR with flag ciphertext → plaintext flag.

## Files
| Artifact | Purpose |
|----------|---------|
| `chal.py` | Encryption routine `whisper()` |
| `out.txt` | Hex nonce, story ciphertext, flag ciphertext |

## Flag Extraction Steps
| Step | Action | Result |
|------|--------|--------|
| 1 | Load hex lines | `nonce`, `C_msg`, `C_flag` bytes |
| 2 | Keystream = `C_msg ⊕ P_msg` | Full keystream bytes |
| 3 | Flag = `C_flag ⊕ keystream` | Plaintext flag |
| 4 | Output | `sunctf25{...}` |

## Exploit Script
```python
with open('out.txt') as f:
    iv_hex, c_msg_hex, c_flag_hex = f.read().splitlines()
msg_ct = bytes.fromhex(c_msg_hex)
flag_ct = bytes.fromhex(c_flag_hex)
message = (b'The streets of New Eridu '  # known narrative
           b'hummed with neon life, untouched by the chaos of the Hollows. Proxies whispered through back alleys, '
           b'chasing commissions that bordered on myth and madness. One night, '
           b'a Hollow surged open near Sixth Street... ')
keystream = bytes(c ^ p for c, p in zip(msg_ct, message))
flag = bytes(c ^ k for c, k in zip(flag_ct, keystream))
print(flag.decode())
```

## Pitfalls
| Issue | Fix |
|-------|-----|
| Mismatch lengths | Truncate to shortest during XOR |
| Missing full plaintext | Need accurate known text portion |
| Nonce confusion | Ensure nonce reused (confirm in file) |

## Mitigation
| Control | Purpose |
|---------|---------|
| Unique nonce per message | Prevent keystream reuse |
| AEAD mode (ChaCha20-Poly1305) | Authenticity + safe reuse checks |
| Monitoring for repeated IV | Detect cryptographic misuse |

## Indicators
Identical nonce with multiple ChaCha20 invocations; large known narrative shipped alongside unknown blob.

## Final Flag
`sunctf25{m4yb3_s0m3_k3y_d1ff3r3nc3_1snt_s0_b4d_4ft3r4LL}`
