# PKI tutorial

A hands-on course in public key infrastructure: concepts first, then a lab with `openssl` for every
lesson, then a short quiz. Each lesson is about 15 minutes of reading plus a lab.

| # | Lesson | Status |
|---|---|---|
| 1 | [The problem, and the building blocks: hashes, key pairs, signatures, session keys, (EC)DHE](lesson-01-building-blocks.md) | in progress |
| 2 | Certificates: what's inside an X.509 certificate, and why one exists at all | |
| 3 | Chains: your own root CA, an intermediate CA, verifying a chain by hand | |
| 4 | TLS and mutual TLS: the handshake, devices proving identity to each other | |
| 5 | Revocation: CRL and OCSP | |
| 6 | Rotation and expiry | |
| 7 | Device identity: keys in a TPM or secure element | |

**Setup:** any shell with OpenSSL 3.x (`openssl version`). On Windows, Git Bash includes it.
Labs work in `~/pki-lab/<lesson>`.

**Books:** *Bulletproof TLS and PKI* (Ivan Ristić) as the main text; *Serious Cryptography*
(Jean-Philippe Aumasson) for the math; RFC 5280 for the certificate format.
