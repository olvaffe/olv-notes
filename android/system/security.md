# Android Security

## Keystore

- <https://developer.android.com/privacy-and-security/keystore>
- components
  - `system/security/keystore2` provides `keystore2` daemon
  - `hardware/interfaces/security/keymint` defines keymint hal
  - `system/core/trusty/keymint` provides a keymint impl that uses trusty tee
  - there may also be keymint impl that uses strongbox
    - strongbox requires a separate security chip
