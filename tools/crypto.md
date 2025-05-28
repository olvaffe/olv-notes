Cryptography
============

## Cryptography

- <https://en.wikipedia.org/wiki/Cryptography>
- cryptography
  - symmetric-key cryptography
  - public-key (asymmetric) cryptography
- hash function
  - non-cryptographic
  - cryptographic
- common encryption algorithms
  - symmetric
    - DES, 1977
    - Blowfish, 1993
    - AES, 2001
    - ChaCha20, 2008
  - asymmetric
    - RSA, 1977
    - DSA, 1994
    - ECDSA, 2000
    - EdDSA, 2011 (ED25519)
  - authenticated (which combines symmetric with mac)
    - AES-GCM
    - ChaCha20-Poly1305
  - modes of operation (of block ciphers)
    - a block cipher is a symmetric-key encryption that operates on
      fixed-length blocks
    - a mode of operation is an algorithm to encrypt multiple blocks, turning
      a block cipher into a stream cipher
    - ECB, block-by-block
    - CBC, each block is XORed with the prior encrypted block first
    - CTR, each block is XORed with the encrypted counter
    - XTS
- key exchange algorithms
  - DH, 1976 (Diffie–Hellman)
    - A and B publicly agree `x` and `p`
    - A picks `a` privately
    - B picks `b` privately
    - A sends the result of `x^a mod p` to B
    - B sends the result of `x^b mod p` to A
    - both A and B use the result of `x^(ab) mod p` as the secret key
  - ECDH (Elliptic-curve Diffie–Hellman)
- e.g.,
  - two parties use key exchange to generate a shared secret key over an
    inscure connection
  - two parties use the shared secret key and symmetric-key encryption to
    create a secure connection
  - one party authenticates the other using asymmetric-key encryption, by
    encrypting a challenge using the other's public key
- key derivation function (KDF)
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
  - Argon2, 2015


## Hash Functions

- common hash functions
  - non-cryptographic
    - CRC-32
    - FNV-1a
    - SipHash, 2012
    - MurmurHash3, 2012
    - xxHash3, 2019
  - cryptographic
    - MD5, 1992
    - SHA-1, 1995
    - SHA-2, 2001 (SHA-256, SHA-512, etc.)
    - SHA-3, 2016
    - BLAKE3, 2020
- MAC
  - the purpose is to check message authenticity and integrity
  - HMAC: `hash(message + shared secret)`

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
