# Lesson 1: the problem PKI solves, and its building blocks

## The problem

Two parties who have **never met** (a phone and a server, say) want to talk safely over a network an
attacker can see and change. They need three things:

| Need | Meaning | Tool |
|---|---|---|
| **Confidentiality** | nobody else can read it | encryption |
| **Integrity** | nobody changed it in transit | hash + signature |
| **Authenticity** | it really came from who it claims | signature |

**Symmetric encryption** (AES) is fast, but both sides need the **same secret key**. How do you get
that key to someone you've never met without an attacker seeing it? That's the **key distribution
problem.**

## Asymmetric cryptography: the key pair

- **Private key:** never leaves its owner.
- **Public key:** can be given to anyone.
- **Two uses:**
  - **Signing:** the owner signs with the **private** key, and anyone can check the signature with the
    **public** key. This proves *who* sent the message and that it wasn't changed.
  - **Key exchange:** two sides combine their key pairs to agree on a shared AES key without ever
    sending it. The fast AES encryption then takes over.

## Hashes

A hash (SHA-256) is a fixed-size fingerprint of any data:
- **One-way:** you can't get the data back from the hash.
- **Change one bit** and the hash changes completely.
- **Collision-resistant:** you can't find two inputs with the same hash.
- **What gets signed is the hash,** not the whole message.

## The gap that's left

When you receive "the server's public key," how do you know it's really the server's and not an
attacker's? An attacker in the middle can hand you **their** key. **PKI exists to close that gap:**
someone you already trust signs a statement that says "this public key belongs to this name." That
statement is a **certificate** (lesson 2).

---

## Lab 1 (Git Bash or any shell with OpenSSL 3)

```sh
mkdir -p ~/pki-lab/lesson1 && cd ~/pki-lab/lesson1

# 1. Hash: fingerprint a file, change one character, compare
echo "transfer 100 to bob" > msg.txt
openssl dgst -sha256 msg.txt
echo "transfer 900 to bob" > msg2.txt
openssl dgst -sha256 msg2.txt

# 2. Key pair: an elliptic-curve private key, then its public half
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out alice.key
openssl pkey -in alice.key -pubout -out alice.pub
cat alice.pub

# 3. Sign with the private key, verify with the public key
openssl dgst -sha256 -sign alice.key -out msg.sig msg.txt
openssl dgst -sha256 -verify alice.pub -signature msg.sig msg.txt

# 4. Tamper: verify the same signature against the changed message
openssl dgst -sha256 -verify alice.pub -signature msg.sig msg2.txt
```

**What to notice:**
- **Step 1:** one character changed, and the hash is completely different.
- **Step 3:** prints `Verified OK`.
- **Step 4:** prints `Verification failure`.

## Quiz

1. Alice signs a message. Which key does **she** use, and which key does **Bob** use to check it?
2. Why do we sign the **hash** of a message instead of the whole message?
3. Bob got "Alice's public key" from an email, and the signature verifies. Is Bob sure the message
   came from Alice? Why or why not?

## My lab output and answers

_(fill in)_

## Questions and answers

### Q: How do public keys get exchanged?
A public key **doesn't need to be secret**; it can travel in the open. The real problem is
**authenticity**: "is this really Alice's key, or an attacker's?" Three ways to answer it:

1. **Out of band:** get the key, or compare its fingerprint, over a second trusted channel. Example:
   SSH's first-connection "are you sure?" prompt (*trust on first use*), or copying `id_rsa.pub` into
   `authorized_keys`. Doesn't scale to millions of strangers.
2. **Pre-installed:** baked in at the factory or install time. Examples: the ~150 root CA keys in every
   OS and browser; an update server's key burned into device firmware (how OTA images are verified).
   Changing it needs a software update.
3. **Inside a certificate (PKI):** the key arrives over the network with a name, **signed by a CA**.
   You don't trust the key, you trust the CA's signature on it, which you can check because you
   already have the CA's key (method 2). Like a passport: you believe the name because you trust the
   issuing government. This is what the server sends in the TLS handshake.

**PKI = methods 2 + 3:** a few pre-installed root keys, and every other key arrives in a certificate
signed by one of them, directly or through a chain.

### Q: A middlebox makes its own key pairs, one facing the client and one facing the server, and decrypts both ways. Does that defeat TLS?
(From my SafeMedia/Clouseau work.) This is a **TLS-intercepting proxy**, the same design as corporate
"SSL inspection" products and as the Varuth proxy (two-sided TLS termination).

- **Toward the server** it's easy: most servers don't check who the client is, so the box just
  connects as an ordinary client.
- **Toward the client** is where PKI stops it: the box must present a certificate for the real server
  name, signed by a CA the client trusts. It can't get one from a public CA, so the browser shows a
  certificate warning.
- **It works only if** the client trusts the box's own root CA (installed in the trust store, as
  enterprises do on managed laptops) or the user clicks through the warning.
- **Mutual TLS defeats it on the server side:** if the server demands a client certificate, the box
  can't produce it without the client's private key.

**Lesson:** PKI doesn't stop someone from making keys. It stops anyone from making keys **that the
other side will believe**.

### Q: Browsers send "public keys" in the handshake, and the other end decrypts traffic with them. What is that?
Two different mechanisms, depending on the TLS version:

- **RSA key exchange (SSL 3.0 to TLS 1.2):** the client makes a random secret, **encrypts it with
  the server's public key** and sends it; the server decrypts it with its private key. Anyone holding
  the **server's private key** can decrypt recorded traffic later (Wireshark can do this when given
  the key). That's how passive decryption worked when you owned the server.
- **(EC)DHE key exchange (optional in TLS 1.2, the only option in TLS 1.3):** each side sends a fresh
  **ephemeral public key** (the "key share"), and both compute the same session key from them. The
  server's long-term private key is only used to **sign**, so stealing it later doesn't decrypt old
  traffic. That property is **forward secrecy**, and it's why passive decryption no longer works:
  you have to be an active proxy (previous question).

So the public keys you saw browsers send were most likely (EC)DH key shares, or the RSA-encrypted
secret, depending on the era. Lesson 4 covers the handshake in detail.

### Q: Back then it was called SSL. Is that different from TLS?
Same protocol family, renamed. Netscape built **SSL 2.0 (1995) and SSL 3.0 (1996)**; the IETF took it
over and renamed it **TLS 1.0 (1999)**, which is essentially SSL 3.1. Then TLS 1.1 (2006), 1.2 (2008)
and 1.3 (2018). All SSL versions and TLS 1.0/1.1 are now deprecated as insecure, but people still say
"SSL certificate". **HTTPS** never changed: it is HTTP carried inside SSL/TLS, on port 443.

In the SSL era, **RSA key exchange** was the norm (DHE existed but was rarely used), so holding the
server's private key was enough to decrypt recorded traffic passively.
