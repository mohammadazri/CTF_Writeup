Here’s a polished, complete write-up for the **“Resemblance” ChaCha20 challenge**, including the recovered flag:

---

# **CTF Write-up: Resemblance (Crypto)**

**Points:** 150
**Author / Team:** warlocksmurf
**Challenge Description:**

> Sir Cha-Cha never liked reusing stuff anyway

**Challenge Files:**

* `chal.py` – Python script that encrypts a message and a flag using ChaCha20.
* `out.txt` – Output containing the nonce, encrypted message, and encrypted flag.

---

## **1️⃣ Challenge Analysis**

The provided Python script `chal.py` uses **ChaCha20**, a stream cipher, to encrypt:

```python
enc_msg = whisper(message, key, iv)
enc_flag = whisper(flag, key, iv)
```

Key observations:

1. The **same key and nonce (`iv`) are reused** for both the message and the flag.
2. Stream ciphers XOR the plaintext with a deterministic keystream. Reusing the keystream is equivalent to a **two-time pad**, which is cryptographically unsafe.
3. This allows us to compute the keystream from the known plaintext message and decrypt the flag:

$$
keystream = ciphertext_{message} \oplus plaintext_{message}  
$$

$$
flag = ciphertext_{flag} \oplus keystream
$$

---

## **2️⃣ Exploit Strategy**

1. Extract the **ciphertext** and **nonce** from `out.txt`.
2. Use the **known plaintext message** to recover the keystream.
3. XOR the keystream with the flag’s ciphertext to reveal the flag.

---

## **3️⃣ Python Exploit Script**

```python
# resemblance.py

# Load out.txt
with open("out.txt") as f:
    iv_hex, enc_msg_hex, enc_flag_hex = f.read().splitlines()

enc_msg = bytes.fromhex(enc_msg_hex)
enc_flag = bytes.fromhex(enc_flag_hex)

# Known plaintext message
message = (
    b"The streets of New Eridu "
    b"hummed with neon life, untouched by the chaos of the Hollows. Proxies whispered through back alleys, "
    b"chasing commissions that bordered on myth and madness. One night, "
    b"a Hollow surged open near Sixth Street... "
)

# Recover keystream
keystream = bytes([c ^ m for c, m in zip(enc_msg, message)])

# Decrypt flag
flag = bytes([c ^ k for c, k in zip(enc_flag, keystream)])

print("Recovered flag:", flag)
```

**Run the script:**

```bash
python resemblance.py
```

---

## **4️⃣ Recovered Flag**

```
sunctf25{m4yb3_s0m3_k3y_d1ff3r3nc3_1snt_s0_b4d_4ft3r4LL}
```

---

## **5️⃣ Key Takeaways**

* **Never reuse the same key and nonce with a stream cipher.**
* Knowing any part of the plaintext allows recovery of other messages encrypted with the same keystream.
* This challenge demonstrates a **classic two-time pad vulnerability** in ChaCha20.

---

If you want, I can also make a **super concise 1-paragraph version** suitable for submitting on CTF platforms—it’s much shorter and straight to the point.

Do you want me to make that version too?
