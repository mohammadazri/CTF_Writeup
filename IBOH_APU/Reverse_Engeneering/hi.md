---
title: HI Hidden Function Invocation
ctf: IBOH_APU
category: Reverse Engineering
points: 150
difficulty: Medium
date: 2025-11-23
flag: BOH25{D1d_u_s0lv3d_it_w17h0u7_CHa7GP7_?_W311_D0n3!}
---

# HI Hidden Function (Concise)

![vector](https://img.shields.io/badge/Vector-PIE%20binary-blue) ![primitive](https://img.shields.io/badge/Primitive-gdb%20symbol%20call-6a5acd) ![impact](https://img.shields.io/badge/Impact-flag%20recovery-critical) ![technique](https://img.shields.io/badge/Technique-hidden%20function%20invoke-orange) ![status](https://img.shields.io/badge/Flag-extracted-success)

## Summary
Locate `secret_func` symbol, resolve relocated runtime address under PIE via `gdb`, invoke directly, flag prints.

## Chain
Enumerate symbols → identify `secret_func` → load under `gdb start` → get runtime address → `call` function pointer → flag.

## Recon
| Item | Value |
|------|-------|
| File | `o.exe` ELF64 PIE |
| Symbol | `secret_func` offset 0x1208 |
| Tools | `file`, `readelf -s`, `gdb` |
| Flag format | `BOH25{...}` |

## Extraction Steps
| Step | Action | Command / Detail | Result |
|------|--------|------------------|--------|
| 1 | Identify PIE | `file o.exe` | Confirms relocation |
| 2 | List symbols | `readelf -s o.exe | grep secret_func` | Offset 0x1208 |
| 3 | Start runtime | `gdb -q ./o.exe -ex start` | Loader done |
| 4 | Resolve address | `info address secret_func` | Runtime addr (e.g. 0x5555...5208) |
| 5 | Invoke function | `call ((void(*)()) secret_func)()` | Flag printed |

## One-Liner
```bash
gdb -q ./o.exe -ex start -ex "info address secret_func" -ex "call ((void(*)()) secret_func)()" -ex quit
```

## Fallback Address Calc
| Step | Detail |
|------|--------|
| Get .text base | `info files` mapping |
| Add offset | base + 0x1208 = runtime address |
| Invoke | `call ((void(*)()) <addr>)()` |

## Pitfalls
| Issue | Fix |
|-------|-----|
| Miss PIE relocation | Run under `start` first |
| Wrong offset added | Use symbol table, not disassembly guess |
| Calling convention confusion | Cast to `void(*)()` pointer |

## Mitigation
| Control | Purpose |
|---------|---------|
| Strip symbols | Hide easy target names |
| Move secrets server-side | Prevent local disclosure |
| Auth checks inside secret func | Require context before printing |
| Obfuscate / rename routines | Increase analyst effort |

## Indicators
Readable symbol `secret_func`; PIE binary with no anti-debug; flag prints directly on raw call.

## Final Flag
`BOH25{D1d_u_s0lv3d_it_w17h0u7_CHa7GP7_?_W311_D0n3!}`



