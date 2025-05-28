Cryptography
============

## Overview

- non-cryptographic hash functions
  - they should be fast, uniform, and a small change in input should result in
    drastic change in output
    - a main application is hash table
  - CRC-32
  - FNV-1a
  - SipHash, 2012
  - MurmurHash3, 2012
  - xxHash3, 2019
- cryptographic hash functions
  - they should be infeasible to find a collision, usually at the cost of
    speed
  - MD5, 1992
  - SHA-1, 1995
  - SHA-2, 2001 (SHA-256, SHA-512, etc.)
  - Poly1305, 2005
  - SHA-3, 2016
  - BLAKE3, 2020
- key exchange algorithms
  - they allow two parties to share a secret over an insecure connection
  - DH, 1976 (Diffie–Hellman)
    - A and B publicly agree `x` and `p`
    - A picks `a` privately
    - B picks `b` privately
    - A sends the result of `x^a mod p` to B
    - B sends the result of `x^b mod p` to A
    - both A and B use the result of `x^(ab) mod p` as the secret key
  - ECDH (Elliptic-curve Diffie–Hellman)
    - A generates a elliptic curve key pair `Apub` and `Apriv`
    - B generates a elliptic curve key pair `Bpub` and `Bpriv`
    - A and B exchange their public keys
    - A computes `Apriv*Bpub`, B computes `Apub*Bpriv`, and they reach the
      same answer which is used as the secret key
- message authentication codes (MAC)
  - the purpose is to validate message authenticity and integrity
  - HMAC: `hash(message + shared secret)`
- symmetric-key cryptography
  - they should be strong and fast
  - DES, 1977
  - Blowfish, 1993
  - AES, 2001
  - ChaCha20, 2008
  - modes of operation (of block ciphers)
    - a block cipher is a symmetric-key encryption that operates on
      fixed-length blocks
    - a mode of operation is an algorithm to encrypt multiple blocks, turning
      a block cipher into a stream cipher
    - ECB, block-by-block
    - CBC, each block is XORed with the prior encrypted block first
    - CTR, each block is XORed with the encrypted counter
    - XTS
- Authenticated encryption with associated data (AEAD)
  - they combine symmetric-key crypto with mac
  - AES-GCM
  - ChaCha20-Poly1305, 2013
- asymmetric-key (public-key) cryptography
  - they support both encryption and signature, but they are slower than
    symmetric crypto
  - RSA, 1977
  - DSA, 1994
  - ECDSA, 2000
  - EdDSA, 2011 (ED25519)
- key derivation functions (KDF)
  - it is used to make passphrase stronger against brute-force attack
    - it is also used for passphrase hashing
  - `DK = KDF(key, salt, iterations)`
    - `DK` is the derived key
    - `KDF` is the function
    - `key` is the original key, such as a passphrase
    - `salt` is a random number
    - `iterations` is the number of iterations
      - it makes brute-force attack more expensive
  - bcrypt, 1999
  - PBKDF2, 2000
  - scrypt, 2009
  - HKDF (HMAC-based KDF), 2010
  - Argon2, 2015

## Secure Connection

- two parties want to establish a secure connection on top of open connection
- authentication with asymetric-key cryptography
  - one party verifies the signed cert of another party
  - or, one party encrypts a challenge using another party's public key
- secret key exchange
  - two parties use key exchange to generate a shared secret key
  - or, one party shares the secret key encrypted by another party's public
    key
- session key derivation
  - two parties independently derive the same session key
- message encryption
  - two parties use the same session key to encrypt future messages
- TLS 1.3
  - establish secure connection for authentication
    - client generates an elliptic curve key pair for ECDH
    - clients sends its pub key to server
    - server generates an elliptic curve key pair for ECDH
    - server sends its pub key to client
    - both compute the same value using ECHD
    - both derive the same key using HKDF
    - all messages are encrypted from this point on
  - authentication
    - server signs the cert and generates a signature
    - server sends the cert and the signature to client
    - client verifies the cert and the signature
      - the cert is signed by CA so it cannot be forged
      - the signature proves that the server has the private key
  - update the encryption key
    - both derive a new key using HKDF
    - all messages are encrypted using the new key from this point on

## Key File Formats

- ASN.1 is an IDL for defining data structures
  - it is a language to describe a data structure
- DER is an encoding format for ASN.1
  - it encodes a data structure into a binary format
- PEM is a text format for DER plus metadata
  - header: `-----BEGIN foo-----`
  - data: base64-encoded DER binary data
  - footer: `-----END foo-----`
- X.509
  - <https://datatracker.ietf.org/doc/html/rfc5280>
  - the data structure is described in ASN.1 and consists of
    - Certificate
      - Version Number
      - Serial Number
      - Signature Algorithm ID
      - Issuer Name
      - Validity period
        - Not Before
        - Not After
      - Subject name
      - Subject Public Key Info
        - Public Key Algorithm
        - Subject Public Key
      - Issuer Unique Identifier (optional)
      - Subject Unique Identifier (optional)
      - Extensions (optional)
        - ...
    - Certificate Signature Algorithm
    - Certificate Signature
  - it can thus be encoded as DER or PEM
- PKCS #8
  - <https://datatracker.ietf.org/doc/html/rfc5208>
  - the data structure is described in ASN.1 and consists of
    - version
    - privateKeyAlgorithm
    - privateKey
    - attributes
  - it can thus be encoded as DER or PEM
- PKCS #12
  - <https://datatracker.ietf.org/doc/html/rfc7292>
  - the data structure is described in ASN.1 and consists of
    - too complex to list
    - but it is essentially an archive format and is often used to store an
      x509 cert and a pkcs8 privkey
      - an alternative is to concatenate an x509 cert and a pkcs8 privkey in
        PEM format
  - it can thus be encoded as DER or PEM
- PKCS #7
  - <https://datatracker.ietf.org/doc/html/rfc2315>
  - the data structure is described in ASN.1 and consists of
    - contentType
    - content
    - iow, it is content plus metadata
  - it can thus be encoded as DER or PEM

## OpenSSL

- CLI
  - symmetric encryption
    - `openssl enc` encrypts/decrypts a file
      - `-aes256` uses AES-256-CBC
      - `-d` decrypts instead of encrypts
  - asymmetric encryption
    - `openssl genpkey` generates a private key
      - this is preferred over `genrsa` or `gendsa`
      - `-algorithm RSA` uses RSA
      - `-algorithm EC` uses ECDSA (and requires params)
      - `-algorithm ED25519` uses EdDSA (ED25519)
    - `openssl pkey` processes a private key
      - this is preferred over `rsa`, `dsa`, or `ec`
      - `-in <pkey> -noout -text` shows the key in text form
    - `openssl pkeyparam` processes key params
      - this is preferred over `dsaparam`, `dhparam`, and `ecparam`
      - `-in <param> -noout -text` shows the params in text form
    - `openssl pkeyutl` uses a private key
      - this is preferred over `rsautl`
  - hash
    - `openssl dgst` hashes a file
      - `-sha256` is the default
  - kdf
    - `openssl kdf` derives a key
  - mac
    - `openssl mac` calculates MAC for a message
  - PKCS
    - `openssl pkcs7`
    - `openssl pkcs8`
    - `openssl pkcs12`
  - cert
    - `openssl req` generates a PKCS #10 cert request
    - `openssl verify` verifies a cert
    - `openssl x509` processes a cert
