# Project 1 - Recovering plaintext from multiple ciphertexts sharing a keystream

**Goal**

This short report describes the method and results of a script that recovers plaintexts from a set of ciphertexts encrypted with the *same keystream* (keystream reuse — a critical mistake for stream ciphers / OTP). The objective is to exploit the relation `ct_i ⊕ ct_j = pt_i ⊕ pt_j` to recover parts of the keystream and reconstruct plaintexts, especially a target ciphertext.

**Method** `Threshold`(main idea) & `Space-trick`(side idea)

1. Convert ciphertexts from hex to bytes.
2. For every pair of ciphertexts `(i, j)` compute `x = ct_i ⊕ ct_j`. If `x[pos]` is an alphabetic character (A–Z, a–z) or space, increment a vote counter for `ct_i` at position `pos`.
3. For each ciphertext `i`, if `votes(pos) ≥ threshold` assume `pt_i[pos] = ' '` and derive `key[pos] = ct_i[pos] ⊕ 0x20`. The bytes found this way form a `partial_key`.
4. Use `partial_key` to decrypt all ciphertexts; unknown positions print as `?`.
5. With `space-trick`, for positions not determined by thresholding, the script tests candidates assuming `space` in one ciphertext, checks consistency with others, and selects the candidate key that maximizes valid English-like letters.
6. Apply `manual_hints`(with computing) to guess exact key bytes from known plaintext characters: `key[pos] = ct_i[pos] ⊕ ord(char)`.
7. Again using `manual_hints` (but with human predictions) for perfect solution: `The secret message is: When using a stream cipher, never use the key more than once`

**How to run**

* Adjust the `threshold` parameter in the script (typical values 5–8).
* Run the Python script; outputs show decrypted plaintexts 1..10 and the target (unknown bytes as `?`).
* If necessary, add `manual_hints` to correct specific bytes (recommend using hints from non-target ciphertexts to preserve attack realism).
* In fact, I don't like the way `space-trick` work, because the way it filling the word is too risky. I like the `threshold` because it guess the word more "math-way". But I'm still using the `space-trick` here since its still makes more correct prediction than `threshold`.

**Results & Limitations**

* The threshold method yields reliable recovery where many ciphertexts provide evidence of a space; positions with little evidence remain unknown (`?`).
* Heuristic filling (space-trick) can fill gaps but may introduce errors by selecting incorrect candidate keys.
* Using manual hints derived from the target is effective but effectively injects the answer and is not a purely automated attack.

**Conclusion & Recommendations**

* Keystream reuse is a severe vulnerability: exposing a few ciphertexts can compromise the entire set.
* To improve automation and accuracy, consider: (a) n-gram or dictionary refinements, (b) preventing overwrite of threshold-derived key bytes during filling, and (c) reporting low-confidence positions so an analyst can provide a small number of non-target hints.

**Tools & Reference**

* ChatGPT and Python for code building and fixing code
* With ChatGPT, It suggests the idea of threshold (main idea) and space-trick (side idea) and also part of this README.
* Using Python to build the code, help automating most of tasks like XORing ciphertexts, auto-filling the encrypted bytes, etc.
