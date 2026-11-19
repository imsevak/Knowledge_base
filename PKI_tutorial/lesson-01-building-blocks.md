# Lesson 1: the problem PKI solves, and its building blocks

**Reading alongside:** *Bulletproof SSL and TLS* (Ristić, 1st ed.): ch. 1 "Cryptography" (PDF pp. 27–45), ch. 2 "Key Exchange" and "Authentication" (pp. 58–65). For §8 (interception): ch. 4 "Widespread SSL Interception" (p. 133) and ch. 5 "MITM Certificates" (p. 148). This edition predates TLS 1.3, so §5–7 here are the up-to-date version.

## 1. The problem

Two parties who have **never met** (a phone and a server, say) want to talk safely over a network an
attacker can see and change. They need three things:

| Need | Meaning | Tool |
|---|---|---|
| **Confidentiality** | nobody else can read it | encryption |
| **Integrity** | nobody changed it in transit | hash + signature |
| **Authenticity** | it really came from who it claims | signature |

## 2. Two kinds of cryptography

**Symmetric** (AES): **one key** both locks and unlocks. Very fast, so it's used for the actual data.
The catch: both sides need **the same secret key**. How do two strangers get one without an attacker
seeing it? That's the **key distribution problem.**

**Asymmetric** (RSA, elliptic curves): **a key pair**, two keys that belong together.
- **Private key:** never leaves its owner.
- **Public key:** can be given to anyone. It doesn't need to be secret.
- What one key does, only the other key can check or undo. About **1,000× slower** than AES, so it's
  used only for small jobs: proving identity, and agreeing on a symmetric key.

## 3. Hashes

A hash (SHA-256) is a fixed-size fingerprint of any data:
- **One-way:** you can't get the data back from the hash.
- **Change one bit** and the hash changes completely.
- **Collision-resistant:** you can't find two inputs with the same hash.
- **What gets signed is the hash,** not the whole message (small, fixed size, fast).

## 4. The three keys and their jobs ⭐

This is the core of the lesson. A secure connection uses **three different keys for three jobs**:

| Key | Kind | Lifetime | Where it lives | Its one job |
|---|---|---|---|---|
| **Private key** | asymmetric | long (months to years) | only on the server; secret | **prove identity**: it signs |
| **Public key** | asymmetric (pair of the private key) | same as the private key | inside the certificate; anyone can have it | lets others **check** those signatures |
| **Session key** | symmetric (AES) | **one connection**, then deleted | both ends, in memory only | **encrypts the data** |

**An analogy:**
- **Private key** = your hand that writes your signature. Only you have it.
- **Public key** = the sample of your signature the bank keeps on file. Anyone can compare against it,
  but nobody can sign with it.
- **Session key** = a padlock key made fresh for one conversation, copied to both sides, and destroyed
  afterwards.

**Why three, not one?** Asymmetric keys are slow and can't share a secret on their own; AES is fast
but needs a shared secret. So asymmetric crypto does the small, important jobs (**who are you?** and
**agree on a key**), then AES does the heavy work.

## 5. How two strangers make the same session key without sending it: (EC)DHE

**Paint analogy (Diffie–Hellman):**
1. Everyone agrees on a common color: **yellow** (public).
2. Alice secretly picks **red**, mixes yellow + red = **orange**, and sends orange.
   Bob secretly picks **blue**, mixes yellow + blue = **green**, and sends green.
3. Alice adds her red to Bob's green; Bob adds his blue to Alice's orange.
   **Both get the same brown** (yellow + red + blue).
4. An eavesdropper saw yellow, orange and green, but **can't un-mix paint**, so can't make brown.

**The same thing with real (tiny) numbers.** Public: prime `p = 23`, base `g = 5`.

| | Alice | Bob |
|---|---|---|
| Secret (ephemeral private key) | `a = 6` | `b = 15` |
| Sends (ephemeral public key, the "key share") | `A = 5^6 mod 23 = 8` | `B = 5^15 mod 23 = 19` |
| Computes from the other's share | `19^6 mod 23 = 2` | `8^15 mod 23 = 2` |

Both get **2**, the shared secret, and it was never sent. Real TLS uses numbers hundreds of digits
long, so working backwards from 8 or 19 to 6 or 15 is impossible in practice.

**Reading the name (EC)DHE:**
- **DH** = Diffie–Hellman (the paint trick).
- **EC** = done on an elliptic curve: same idea, smaller keys, faster.
- **E** = **Ephemeral**: brand-new secrets for **every connection**, thrown away afterwards. If someone
  steals the server's long-term private key next year, old recorded traffic still can't be decrypted.
  That property is **forward secrecy.**

The session key (the AES key) is derived from that shared secret.

## 6. Then what is the server's private key for?

(EC)DHE alone has a hole: **Alice doesn't know who sent "green."** An attacker in the middle can do
the paint trick separately with each side and read everything (section 8).

So the server **signs the handshake** (including its key share) **with its long-term private key.**
The client checks that signature with the server's **public key**, which arrived inside the server's
**certificate**. In TLS 1.3 the private key **never encrypts data.** Its only job is to prove "it's
really me making this key share."

## 7. Is (EC)DHE what PKI means?

No. They are two separate pieces that TLS uses together:
- **(EC)DHE** answers: *how do we get a shared secret key?* (secrecy)
- **PKI** (Public Key Infrastructure) answers: *how do I know this public key really belongs to
  `bank.com`?* It is the whole system for vouching for public keys: **certificate authorities (CAs),
  certificates, the trust store of root keys pre-installed in every OS and browser, and revocation.**

### One TLS 1.3 connection, step by step (all three keys)
1. **Client → server:** "hello" + the client's **ephemeral key share** (Alice's orange).
2. **Server → client:** its **ephemeral key share** (Bob's green) + its **certificate** (server's public
   key + name, signed by a CA) + a **signature over the handshake made with its private key.**
3. **Client checks (PKI):** the certificate is signed by a CA it trusts (traced back to a
   pre-installed root), and the name matches. Then it verifies the server's signature with the
   **public key** from that certificate. Now it knows it's really the server.
4. **Both compute the session key** from the two key shares ((EC)DHE).
5. **All data is encrypted with the AES session key.** When the connection closes, the session key and
   the ephemeral secrets are deleted.

## 8. Case study: intercepting TLS (my SafeMedia/Clouseau work)

The idea: a box in the middle makes its own key pairs, one facing the client and one facing the
server, and decrypts traffic both ways. This is a **TLS-intercepting proxy**: the same design as
corporate "SSL inspection" products and as the Varuth proxy.

```
 Browser  ⇄  TLS connection #1  ⇄  [ Clouseau ]  ⇄  TLS connection #2  ⇄  Server
        session key #1                decrypts,           session key #2
                                   inspects, re-encrypts
```

Clouseau runs **two separate TLS connections** and does (EC)DHE with each side, so it holds both
session keys. The question is: **will each side accept it?**

- **Toward the server: easy.** Most servers don't ask who the client is (no client certificate), so
  Clouseau simply connects as an ordinary client.
- **Toward the browser: PKI stops it** (step 3 above). Clouseau doesn't have the real server's private
  key, so it must present **its own** certificate for `bank.com`. Signed by whom? No public CA will
  sign that, so the browser shows a **certificate warning.**
- **It works only if** the browser trusts Clouseau's own root CA (installed into the trust store, as
  companies do on managed laptops) or the user clicks past the warning.
- **Mutual TLS defeats it on the server side:** if the server demands a client certificate, Clouseau
  can't produce one without the client's private key.

**The old passive way (SSL era):** SSL 3.0 to TLS 1.2 usually used **RSA key exchange**: the browser
made a random secret, **encrypted it with the server's public key** and sent it. Anyone holding the
**server's private key** could decrypt recorded traffic later, with no box in the middle. TLS 1.3
removed RSA key exchange (forward secrecy), so today interception must be active, like Clouseau.

**The one-line lesson:** PKI doesn't stop anyone from *making* keys. It stops them from making keys
**the other side will believe.**

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

# 5. (EC)DH: Alice and Bob each make a key pair, then derive the same secret
openssl genpkey -algorithm X25519 -out a.key && openssl pkey -in a.key -pubout -out a.pub
openssl genpkey -algorithm X25519 -out b.key && openssl pkey -in b.key -pubout -out b.pub
openssl pkeyutl -derive -inkey a.key -peerkey b.pub | xxd -p -c 32
openssl pkeyutl -derive -inkey b.key -peerkey a.pub | xxd -p -c 32
```

**What to notice:**
- **Step 1:** one character changed, and the hash is completely different.
- **Step 3:** prints `Verified OK`.
- **Step 4:** prints `Verification failure`.
- **Step 5:** both lines print **the same** value: a shared secret that was never sent.

## Quiz

1. Alice signs a message. Which key does **she** use, and which key does **Bob** use to check it?
2. Why do we sign the **hash** of a message instead of the whole message?
3. Bob got "Alice's public key" from an email, and the signature verifies. Is Bob sure the message
   came from Alice? Why or why not?
4. In TLS 1.3, which key encrypts your data, and what is the server's private key used for?
5. An attacker records today's TLS 1.3 traffic and steals the server's private key next year. Can they
   decrypt the recording? Why or why not?

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

### Q: Can a middlebox with its own key pairs decrypt TLS both ways?
See **section 8** (case study: intercepting TLS).

### Q: Browsers send "public keys" in the handshake. What are they?
Depends on the version (sections 5 and 8): in TLS 1.3 they are **ephemeral (EC)DH key shares**; in the
SSL era it was usually the **RSA-encrypted random secret**.

### Q: Back then it was called SSL. Is that different from TLS?
Same protocol family, renamed. Netscape built **SSL 2.0 (1995) and SSL 3.0 (1996)**; the IETF took it
over and renamed it **TLS 1.0 (1999)**, which is essentially SSL 3.1. Then TLS 1.1 (2006), 1.2 (2008)
and 1.3 (2018). All SSL versions and TLS 1.0/1.1 are now deprecated as insecure, but people still say
"SSL certificate". **HTTPS** never changed: it is HTTP carried inside SSL/TLS, on port 443.
