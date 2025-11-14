Linux Crypto
============

## Algorithms

- `crypto_register_alg` registers a `crypto_alg`
  - `CRYPTO_ALG_TYPE_CIPHER` is for single-block symmetric ciphers
    - `cra_u.cipher` must be initialized
  - `CRYPTO_ALG_TYPE_COMPRESS` is for (de)compressions
    - `cra_u.compress` must be initialized
  - `CRYPTO_ALG_TYPE_AEAD` stands for Authenticated encryption with associated data
    - use `crypto_register_aead` to register a `aead_alg`
  - `CRYPTO_ALG_TYPE_SKCIPHER` stands for symmetric key cipher
    - use `crypto_register_skcipher` to register a `skcipher_alg`
  - `CRYPTO_ALG_TYPE_KPP` stands for key-agreement protocol primitives
    - use `crypto_register_kpp` to register a `kpp_alg`
  - `CRYPTO_ALG_TYPE_ACOMPRESS` stands for asynchronous (de)compressions
    - use `crypto_register_acomp` to register a `acomp_alg`
  - `CRYPTO_ALG_TYPE_SCOMPRESS` stands for synchronous (de)compressions
    - use `crypto_register_scomp` to register a `scomp_alg`
  - `CRYPTO_ALG_TYPE_RNG` stands for random number generators
    - use `crypto_register_rng` to register a `rng_alg`
  - `CRYPTO_ALG_TYPE_AKCIPHER` stands for asymmetric/public key cipher
    - use `crypto_register_akcipher` to register a `akcipher_alg`
  - `CRYPTO_ALG_TYPE_HASH` stands for hashes/digests
  - `CRYPTO_ALG_TYPE_SHASH` stands for synchronous hashes/digests
    - use `crypto_register_shash` to register a `shash_alg`
  - `CRYPTO_ALG_TYPE_AHASH` stands for asynchronous hashes/digests
    - use `crypto_register_ahash` to register a `ahash_alg`
- templated algorhtims
  - `crypto_register_template` registers a `crypto_template`
    - for example, `rsa_pkcs1pad_tmpl`
  - `crypto_register_instance` registers a `crypto_instance`, which contains a
    `crypto_alg`
    - for example, `pkcs1pad_create` calls `akcipher_register_instance`
- algorithm lookups
  - `crypto_alg_lookup` looks up a `crypto_alg`
    - it matches `cra_driver_name` or `cra_name` exactly
  - `crypto_larval_lookup` is a wrapper for `crypto_larval_lookup`
    - it attemps `request_module` when no algorithm is found
  - `crypto_alg_mod_lookup` is a wrapper for `crypto_larval_lookup`
    - it calls `crypto_probing_notify` to register a templated algorithm when
      no algorithm is found
  - `cryptomgr_schedule_probe` spawns `cryptomgr_probe` kthread to register a
    templated algorithm
    - `crypto_lookup_template` looks up a `cryto_template` and calls its
      `create` callback
- usage, using `CRYPTO_ALG_TYPE_AKCIPHER` as an example
  - allocate a tfm handle
    - algorithms are also referred to as transformations (tfm)
    - `crypto_alloc_akcipher` allocates a `crypto_akcipher`
  - prepare a tfm handle
    - `crypto_akcipher_set_priv_key` sets the private key
    - `crypto_akcipher_set_pub_key` sets the public key
  - allocate a request
    - `akcipher_request_alloc` allocates a `akcipher_request`
  - prepare a request
    - `akcipher_request_set_callback`
    - `akcipher_request_set_crypt`
  - submit a request
    - `crypto_akcipher_encrypt`
    - `crypto_akcipher_decrypt`
    - `crypto_akcipher_sign`
    - `crypto_akcipher_verify`

## Asymmetric Keys

- when `CONFIG_CFG80211` is enabled, `regulatory_init_db` initializes the
  regulatory db
  - `CONFIG_CFG80211_REQUIRE_SIGNED_REGDB` is enabled by default
    - `load_builtin_regdb_keys` calls `keyring_alloc` to create a keyring,
      `builtin_regdb_keys`
  - `CONFIG_CFG80211_USE_KERNEL_REGDB_KEYS` is also enabled by default
    - `x509_load_certificate_list` loads `shipped_regdb_certs` to create keys
      in `builtin_regdb_keys`
    - `shipped_regdb_certs` is generated from `net/wireless/certs/`
    - `x509_load_certificate_list` calls `key_create_or_update` to create
      `asymmetric` keys
    - `key_create_or_update` calls `key_type_lookup` to look up
      `key_type_asymmetric` and calls its `preparse` callback
- on init, `asymmetric_key_init` calls `register_key_type` to register
  `key_type_asymmetric`
  - other modules calls `register_asymmetric_key_parser` to register
    `asymmetric_key_parser`, such as `x509_key_parser`
- on key creation, `asymmetric_key_preparse` pre-parses the payload
  - it tries all registered parsers
  - `x509_key_preparse` calls `x509_cert_parse` to parse the payload as a X509
    certificate
    - `x509_check_for_self_signed` checks if the cert is self-signed and
      verifies the signature
    - `public_key_verify_signature`
      - `software_key_determine_akcipher` determines the akcipher
        - RSA signatures usually use `pkcs1pad(rsa)`, where `pkcs1pad` is a
          template and `rsa` is a cipher
      - `crypto_alloc_akcipher` calls `crypto_alloc_tfm` to allocate a
        `crypto_akcipher` of type `crypto_akcipher_type`
      - `akcipher_request_alloc` allocates a request
      - `crypto_akcipher_verify` invokes the verification
        - `pkcs1pad_verify` calls `crypto_akcipher_encrypt` with the child
          algorithm
          - `rsa_enc` for sw encrypt
          - `ccp_rsa_encrypt` if `CONFIG_CRYPTO_DEV_CCP_CRYPTO`
