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

_(questions that come up while studying this lesson)_
