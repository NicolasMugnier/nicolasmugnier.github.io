---
tags: [web3, bip39, passphrases, security]
author: Nicolas Mugnier
categories: web3
title: "The Ultimate Guide to BIP39 Passphrases"
description: "Demystifying BIP39: A Conceptual Breakdown of Passphrase Generation"
image: /assets/img/bip39.png
locale: en_US
---

## Introduction: Why Passphrase Security Matters

In the world of cryptocurrencies and digital assets, your passphrase is the key to your kingdom. Without proper security, your assets could be lost forever. BIP39 passphrases offer a standardized, human-readable way to secure your keys. This article explores how they work, how to generate them, and best practices for validation.

---

## How Does a BIP39 Passphrase Work?

### Entropy, Words, and Checksum

- **Entropy**: The foundation of your passphrase, it's a string of random bits (128-256 bits).
- **Words**: The entropy is mapped to a sequence of words from a standardized 2048-word list.
- **Checksum**: A small piece of data appended to the entropy ensures the passphrase's integrity.

### Why Does Order Matter?

The sequence of words is critical because it directly affects the entropy-checksum mapping. Changing the order will lead to an entirely different key, making it unrecoverable.

---

## Generating a BIP39 Passphrase: Step-by-Step

Here's how a passphrase is created:

1. **Generate Entropy**: Randomly produce 128 to 256 bits.
   ```python
   def generate_entropy(bits: int = 128) -> bytes:
       if bits not in [128, 160, 192, 224, 256]:
           raise ValueError("Unsupported entropy size.")
       return os.urandom(bits // 8)
   ```
2. **Compute Checksum**: Use the first entropy_bits/32 bits of the SHA-256 hash of the entropy. For example checksum length is 4 for a 128 bits entropy (128 / 32).
   ```python
   def calculate_checksum(entropy: bytes) -> int:
       checksum_bits: int = len(entropy) * 8 // 32
       entropy_hash: bytes = hashlib.sha256(entropy).digest()
       checksum: int = int.from_bytes(entropy_hash, "big") >> (256 - checksum_bits)
       return checksum
   ```
3. **Combine Entropy + Checksum**: Append the checksum to the entropy.
   ```python
   entropy_bits: str = bin(int.from_bytes(entropy, "big"))[2:].zfill(len(entropy) * 8)
   checksum_bits: str = bin(calculate_checksum(entropy))[2:].zfill(len(entropy) * 8 // 32)
   combined_bits: str = entropy_bits + checksum_bits
   ```
4. **Map to Words**: Split the result into 11-bit segments and map each segment to a word in the BIP39 wordlist. As the words list contains 2048 words indexed from 0 to 2047, the maximum index 2047 (decimal) = 11111111111 (binary) that's the reason why the result is splitted into 11-bit segments.
   ```python
   mnemonic_words: list[str] = []
   for i in range(0, len(combined_bits), 11):
       word_index: int = int(combined_bits[i:i + 11], 2)
       mnemonic_words.append(wordlist[word_index])
   return mnemonic_words
   ```

### Full Code Example: Python Implementation

```python
import os
import hashlib
import requests
from requests import Response

url: str = 'https://raw.githubusercontent.com/bitcoin/bips/master/bip-0039/english.txt'
response: Response = requests.get(url)
if response.status_code == 200:
    wordlist: list = response.text.splitlines()
else:
    raise RuntimeError("Can not open bip-39 remote file.")

def generate_entropy(bits: int = 128) -> bytes:
    if bits not in [128, 160, 192, 224, 256]:
        raise ValueError("Unsupported entropy size.")
    return os.urandom(bits // 8)

def calculate_checksum(entropy: bytes) -> int:
    checksum_bits: int = len(entropy) * 8 // 32
    entropy_hash: bytes = hashlib.sha256(entropy).digest()
    checksum: int = int.from_bytes(entropy_hash, "big") >> (256 - checksum_bits)
    return checksum

def convert_entropy_to_mnemonic(entropy: bytes) -> list:
    entropy_bits: str = bin(int.from_bytes(entropy, "big"))[2:].zfill(len(entropy) * 8)
    checksum_bits: str = bin(calculate_checksum(entropy))[2:].zfill(len(entropy) * 8 // 32)
    combined_bits: str = entropy_bits + checksum_bits

    mnemonic_words: list[str] = []
    for i in range(0, len(combined_bits), 11):
        word_index: int = int(combined_bits[i:i + 11], 2)
        mnemonic_words.append(wordlist[word_index])
    return mnemonic_words

def generate_mnemonic(bits: int = 128) -> str:
    entropy: bytes = generate_entropy(bits)
    mnemonic_words: list[str] = convert_entropy_to_mnemonic(entropy)
    return " ".join(mnemonic_words)

mnemonic_phrase: str = generate_mnemonic(256)
print("Recovery Phrase:", mnemonic_phrase)
```

---

## Validating a BIP39 Passphrase

### Common Risks

- **Wordlist Tampering**: Always verify that the wordlist used matches the official BIP39 standard.
- **Checksum Mismatch**: A passphrase is invalid if the checksum doesn't align with the entropy.
- **Order of Words**: An incorrect order makes the passphrase irrecoverable.

### Steps to Verify a Passphrase

**a) Check the Words Against the BIP39 List**

- Ensure that each word is present in the official list of 2048 words.
- Be cautious with spelling: even a single mistake makes the passphrase invalid.

**b) Reconstruct the Entropy Bits**

- Each word corresponds to an index (0 to 2047) in the BIP39 list.
- Convert each word to its binary value (11 bits per word).
- Concatenate these values to form a binary sequence.

**c) Verify the Checksum**

- The checksum corresponds to the last bits of the binary sequence.
- Compute the checksum by hashing the entropy bits using SHA-256.
- Compare the computed checksum with the one extracted from the passphrase.
- If they do not match, the passphrase is invalid.

### Validation Example: Python

```python
import hashlib
import requests
from requests import Response

url: str = 'https://raw.githubusercontent.com/bitcoin/bips/master/bip-0039/english.txt'
response: Response = requests.get(url)
if response.status_code == 200:
    BIP39_WORDS: list = response.text.splitlines()
else:
    raise RuntimeError("Can not open bip-39 remote file.")

def verify_passphrase(phrase):
    words = phrase.split()
    if len(words) not in [12, 15, 18, 21, 24]:
        return "Invalid passphrase : incorrect words count"

    if not all(word in BIP39_WORDS for word in words):
        return "Invalid passphrase : word is no part of BIP39 list."

    binary = ''.join(format(BIP39_WORDS.index(word), '011b') for word in words)

    entropy_length = len(binary) - (len(binary) // 33)
    checksum_length = len(binary) - entropy_length

    entropy_bits = binary[:entropy_length]
    checksum_bits = binary[entropy_length:]

    entropy_bytes = bytes(int(entropy_bits[i:i+8], 2) for i in range(0, len(entropy_bits), 8))
    hash_checksum = bin(int(hashlib.sha256(entropy_bytes).hexdigest(), 16))[2:].zfill(256)[:checksum_length]

    if checksum_bits != hash_checksum:
        return "Invalid passphrase : checksum missmatch."

    return "Successful passphrase validation"

phrase = "abandon ability able about above absent absorb abstract absurd abuse access accident"
print(verify_passphrase(phrase))
```

---

## Collisions in Passphrases: How Secure Are They?

A **collision** occurs when two passphrases map to the same underlying private key. Understanding the number of possibilities and the risks of generating an existing passphrase is crucial for assessing the security of BIP39.

### Number of Possibilities

The number of possible BIP39 passphrases depends on the length of the entropy used during generation:

- **128 bits**: $$2^{128}\approx3.4\times10^{38}$$ possibilities.
- **256 bits**: $$2^{256}\approx1.15\times10^{77}$$ possibilities.

For a 12-word passphrase (128 bits of entropy + 4 bits of checksum), the number of possible combinations is:

$$2^{128+4} = 2^{132}\approx5.4\times10^{39}$$

For a 24-words passphrase (256 bits of entropy + 8 bits of checksum), the number of possible combinations is:

$$2^{256 + 8} = 2^{264} \approx2.3\times10^{79}$$

### Risk of Generating an Existing Passphrase

Using random generation, the risk of creating a collision is incredibly low, thanks to the vast number of possibilities.

**Birthday Paradox Approximation**:
The probability P of a collision after generating N passphrases can be approximated by:

$$P\approx 1 - e^{-\frac{N^2}{2M}}$$

Where:

- M = $$2^{128}$$ or $$2^{256}$$ (total possibilities).
- N = Number of generated passphrases.

Even with $$N=10^{18}$$ (a quintillion passphrases), the probability of collision for $$M=2^{128}$$ is negligible.

### Why the Order of Words Matters

The order of words significantly impacts the entropy and checksum, making it almost impossible to generate the same passphrase accidentally:

- Reordering even two words results in a completely different key.
- This property strengthens collision resistance.

### Practical Risks of Collisions

While theoretically possible, the chances of a collision are so astronomically low that they are negligible in practice. The real risks lie in poor implementations or predictable entropy sources, such as:

- Using non-random or biased entropy (e.g., timestamp-based random generation).
- Failing to use a secure random number generator.

By adhering to the BIP39 standard and using cryptographically secure libraries, you can avoid these issues.

### Conclusion on Collisions

BIP39 passphrases provide a level of security that is practically immune to collisions when implemented correctly. While collisions are theoretically possible, the enormous key space ensures they remain an irrelevant concern for real-world applications.

---

## Conclusion and Resources

In this article, we explored the principles behind generating a BIP39 passphrase and detailed the various steps involved in its creation. We also examined the risk of collision, shedding light on the immense number of possibilities these passphrases provide. Furthermore, we discussed how to validate a passphrase to ensure its integrity and compliance with the BIP39 standard.

It is important to note that this article is purely an intellectual exercise and does not aim to provide production-ready code. Its purpose is to deepen understanding of the underlying mechanisms and inspire curiosity about the fascinating world of cryptographic security.

**Additional Resources**:

- [BIP39 Official Documentation](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
- [Gist: Code Examples for Generating Passphrases](https://gist.github.com/NicolasMugnier/a71ca2183528092bfcabcd2aeb72f9fe)
- [Birthday Problem](https://en.wikipedia.org/wiki/Birthday_problem)
