Perfect! You’ve successfully recovered the flag:

```
sunctf25{u6ly_c55_c0l0r5}
```

Here’s a structured write-up for your **Packet Palette – “Auraaa” challenge**:

---

## **CTF Write-up: Packet Palette – Auraaa (Misc)**

**Challenge:**

* Title: *Packet Palette*
* Points: 80
* Hint: “Let’s do some network traffic analysis…”
* Provided: `traffic_latest.pcapng` Wireshark capture.

---

### **Step 1: Inspect the PCAP**

We start by checking the traffic in the capture:

```bash
tshark -r traffic_latest.pcapng
```

Observations:

* All traffic is TCP between `172.18.241.1` → `172.18.240.1` on port 5000.
* Multiple HTTP requests and responses:

  * `/dummy.js` → small JS file
  * `/image.jpg` → large JPEG
  * `/` → HTML page with \~3 KB of content

Hints in the traffic:

* The HTML response contains `<p>` tags with classes `paragraph1` to `paragraph25`.
* CSS colors are defined for each paragraph (e.g., `.paragraph1 { color: #73aaee; }`).
* Challenge description hints:

  * “Colors carry codes”
  * “Hexadecimal isn’t only for programmers”

---

### **Step 2: Identify the data encoding**

The flag is hidden in the **CSS color values**.

* CSS uses **6-digit hex codes** (`#RRGGBB`) for each paragraph.
* Each hex pair can represent **ASCII characters**.
* Example: `#73aaee` → bytes `73 aa ee`.

---

### **Step 3: Extract the colors**

All colors in order:

```
73aaee,75aaee,6eaaee,63aaee,74bb12,
66bb12,32bb12,35bb12,7bbb12,75ddaa,
36ddaa,6cddaa,79ddaa,5fddaa,63ddff,
35ddff,35ddff,5fddff,63ddff,30ddff,
6cddff,30ddff,72ddff,35ddff,7dddff
```

---

### **Step 4: Convert hex to ASCII**

We convert all color values to bytes, then **filter only printable ASCII**:

```python
data = b's\xaa\xeeu\xaa\xeen\xaa\xeec\xaa\xeet\xbb\x12f\xbb\x122\xbb\x125\xbb\x12{\xbb\x12u\xdd\xaa6\xdd\xaal\xdd\xaay\xdd\xaa_\xdd\xaac\xdd\xff5\xdd\xff5\xdd\xff_\xdd\xffc\xdd\xff0\xdd\xffl\xdd\xff0\xdd\xffr\xdd\xff5\xdd\xff}\xdd\xff'
flag = ''.join([chr(b) for b in data if 32 <= b <= 126])
print(flag)
```

* This ignores all non-printable bytes (`\xaa`, `\xdd`, etc.) and recovers the readable flag.

---

### **Step 5: Flag**

```
sunctf25{u6ly_c55_c0l0r5}
```

---

### **Summary / Notes**

* **Technique:** Steganography in CSS color values
* **Key hints:**

  * Text content of HTML misleading
  * Colors encode the flag
  * Filter printable ASCII from raw hex bytes
* **Tools used:** `tshark`, Python

---

This is a complete and professional write-up for submission.

---

If you want, I can also **make a 3–4 sentence “CTF report summary” version** suitable for your write-up doc or Hackathon notes.

Do you want me to do that?
