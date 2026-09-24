# PixelVault

**Password-protected data hiding in images using AES-256-GCM encryption and LSB steganography.**

🔗 **Live demo:** https://sheshadrimath.github.io/pixelvault/

A cryptography micro project that combines two layers of protection: the message is first
**encrypted** so it cannot be read, then **hidden** inside an ordinary image so it is not noticed.
Everything runs inside the browser — the message and password never leave the user's device.

---

## Problem statement

Encryption makes a message unreadable, but it does not hide the fact that a secret exists. In
environments where encrypted files themselves attract attention, the act of encrypting can be as
incriminating as the content. Steganography solves the opposite half of the problem: it conceals the
existence of the message but offers no protection once the hidden data is found.

PixelVault combines both. Even if steganalysis detects that data is hidden in the image, the payload
remains AES-256 encrypted and unreadable without the password.

---

## Features

- **AES-256-GCM encryption** with a password-derived key
- **PBKDF2 key derivation** (HMAC-SHA256, 200,000 iterations, 16-byte random salt)
- **Authenticated encryption** — a wrong password fails cleanly on the GCM tag check instead of
  producing garbage output
- **LSB steganography** across the R, G and B channels of the image
- **Capacity check** before embedding, with a live usage meter
- **Technical readout** showing the salt, nonce, payload size, pixels touched and the percentage of
  colour values changed
- **Fully client-side** — no server, no upload, works offline
- **Zero dependencies** — no frameworks, no libraries, no build step

---

## How it works

### 1. Encryption

```
password + random salt (16 B)
   → PBKDF2-HMAC-SHA256, 200,000 iterations
   → 256-bit AES key
   → AES-256-GCM with a random 12-byte nonce
   → ciphertext + 16-byte authentication tag
```

The payload written into the image is:

```
[4-byte length][16-byte salt][12-byte nonce][ciphertext + tag]
```

The salt and nonce are not secret and are stored with the payload. They are regenerated randomly on
every run, so encrypting the same message twice with the same password produces completely different
output.

### 2. Steganography

Each colour value in a pixel is one byte (0–255). The least significant bit is replaced with one bit
of the payload:

```
new = (old AND 254) OR bit
```

Example — hiding bit `0` in the value 87:

```
  87 = 01010111
 254 = 11111110
 AND = 01010110 = 86
```

The value moves by at most 1, which is imperceptible. The first 32 bits store the payload length so
the extractor knows where to stop, and also act as a sanity check that rejects images with nothing
hidden in them.

**Capacity:** `(width × height × 3 − 32) ÷ 8` bytes. A 1280×720 image holds roughly 337 KB.

---

## Technology

| Layer | Implementation |
|---|---|
| Cryptography | Web Crypto API (`crypto.subtle`) — native, audited browser implementation |
| Randomness | `crypto.getRandomValues()` (CSPRNG) |
| Pixel access | Canvas API (`getImageData` / `putImageData`) |
| File handling | FileReader API |
| UI | Plain HTML, CSS and JavaScript |
| Dependencies | None |
| Backend | None, by design |

No custom cryptography was written. The browser's vetted primitives are used throughout, following
the principle that hand-rolled crypto is a liability.

---

## Running locally

Requires a local server — the Web Crypto API is blocked on `file://` URLs.

1. Clone or download this repository.
2. Open the folder in VS Code.
3. Install the **Live Server** extension.
4. Right-click `index.html` → **Open with Live Server**.

Or simply open the [live demo](https://sheshadrimath.github.io/pixelvault/).

---

## Usage

**To hide a message**
1. Select a cover image (PNG or JPG, no transparency).
2. Type the message and a password.
3. Click *Encrypt and hide* — a stego PNG downloads automatically.

**To reveal a message**
1. Switch to the *Reveal a message* tab.
2. Select the stego PNG and enter the password.
3. Click *Extract and decrypt*.

⚠️ The output **must stay a PNG**. Converting it to JPG destroys the hidden data.

---

## Testing

| # | Test | Result |
|---|---|---|
| 1 | Hide then reveal with the correct password | ✅ Message recovered exactly |
| 2 | Reveal with a wrong password | ✅ GCM tag check fails, no output produced |
| 3 | Reveal from an image with nothing hidden | ✅ Rejected by the length-header check |
| 4 | Stego PNG re-saved as JPG, then revealed | ✅ Fails — confirms lossless format is required |
| 5 | Same message + password encrypted twice | ✅ Different salt, nonce and output each time |
| 6 | Visual comparison of cover vs stego | ✅ No perceptible difference |

**Sample run** — 14-byte message in a 1280×720 image:

```
message           : 14 bytes
ciphertext + tag  : 30 bytes
payload embedded  : 58 bytes (496 bits)
pixels touched    : 166 of 921,600
values changed    : 246 of 496 (49.6%)
capacity used     : 0.02%
PBKDF2 + AES time : 52 ms
```

The 49.6% figure matches theory: a random bitstream flips roughly half the least significant bits it
overwrites.

---

## Security notes

- The message and password are never transmitted. GitHub Pages serves the HTML file, but the
  encryption happens entirely in the visitor's browser.
- The derived key exists only in memory. Nothing is written to localStorage, cookies or the URL.
- Salt and nonce are freshly generated for every operation. A nonce is never reused with the same key.
- AES-256-GCM provides confidentiality, integrity and authenticity together, so tampering with the
  stego image is detected rather than silently ignored.

---

## Limitations

This is a teaching implementation, and it is honest about what it does not do:

- **Sequential LSB embedding is detectable.** Statistical steganalysis (chi-square, RS analysis) can
  flag LSB-modified images, particularly at high capacity usage. Encryption means detection does not
  equal disclosure, but the concealment itself is not robust.
- **No robustness to processing.** Any recompression, resizing, cropping or filtering destroys the
  payload.
- **Password strength is the weakest link.** AES-256 cannot be brute-forced; a short password can be.
  PBKDF2's 200,000 iterations slow guessing but cannot rescue a weak password.
- **Large images are processed on the main thread**, so very large files briefly block the UI.

---

## Future improvements

- Password-seeded pseudo-random pixel selection instead of sequential embedding
- Adaptive embedding that favours textured regions over smooth areas
- Support for hiding arbitrary files, not just text
- A Web Worker so large images do not block the interface
- A built-in steganalysis view (LSB plane visualisation, histogram comparison)

---

## References

1. NIST FIPS 197 — Advanced Encryption Standard (AES)
2. NIST SP 800-38D — Galois/Counter Mode (GCM) of Operation
3. RFC 8018 — PKCS #5: Password-Based Cryptography Specification (PBKDF2)
4. MDN Web Docs — Web Crypto API, Canvas API
5. MITRE ATT&CK — T1027.003, Obfuscated Files or Information: Steganography

---

## Author

**SHESHADRI MATH**
B.Tech — Cyber/Computer Forensics and Counterterrorism, Alliance University
Cryptography micro project, 2026
